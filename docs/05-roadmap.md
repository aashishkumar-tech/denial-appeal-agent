# 5. Roadmap (production build)

## 5.1 Phases (estimates for a 2-developer team)
| Phase | Duration | Owner | Deliverables |
|---|---|---|---|
| **1. Data onboarding (first)** | Weeks 1–4 | Dev A + Dev B | Bring all data onto the platform: repo, local Docker Compose, PostgreSQL + pgvector schema, loaders for public datasets, denial label generator, document store, data checks (see 5.1c) |
| 1b. Platform foundation | Weeks 3–5 | Dev B | Oracle VM, k3s, Argo CD, CI/CD, secrets, backups, monitoring; deploy the data layer to `dev` |
| 2. Rules + ML | Weeks 6–9 | Dev A | Rules engine, features, overturn model, priority score, model registry (MLflow), model report |
| 3. Documents + RAG + Agent | Weeks 9–15 | Dev A | Upload processing, OCR, classifier, extraction, policy index, LLM gateway (OpenAI + Bedrock/Groq fallback), agent tools, letter templates, citation checks |
| 4. API + Dashboard | Weeks 3–16 | Dev B | FastAPI, React dashboard (all screens), login + MFA, roles, per-user data filtering, row-level security, document upload, viewer and side panel |
| 5. Hardening | Weeks 16–19 | Dev A + Dev B | Security review and penetration test (external), load tests, access tests, audit log checks, backups, disaster recovery test |
| 6. Go-live | Weeks 20–21 | Dev A + Dev B | HIPAA sign-off (BAAs for OpenAI and Bedrock/Groq), user training, production release, support process |
| 7. Improve | Ongoing | Dev A + Dev B | Retrain on real outcomes, template improvements from reviewer edits, new payers |

**Team split:** Dev A = data, ML, AI/agent (Python). Dev B = API, security, React dashboard.
**Ownership:** the 2 developers own everything — build, testing, deployment, support and sign-off of technical decisions.
**Capacity note:** with 2 developers, ~21 weeks is realistic. To go faster, launch with fewer screens (queue, case detail, appeal review, upload/viewer, admin) and add analytics after go-live.

## 5.1a Minimum-investment approach
| Area | Low-cost choice |
|---|---|
| Primary LLM | OpenAI small model (e.g. `gpt-4o-mini` class) for extraction and classification; larger model only for appeal letters |
| Fallback LLM | Llama 3.3 70B via Groq or Bedrock (pay per use, no servers) |
| OCR | Tesseract (free) first; paid OCR (e.g. AWS Textract) only for pages where Tesseract confidence is low |
| Embeddings | Free open-source embedding model (e.g. `BAAI/bge-small-en-v1.5`) running on CPU, stored in pgvector |
| ML | XGBoost / LightGBM on CPU (no GPU) |
| Database + vectors | One PostgreSQL + pgvector instance (no separate vector DB) |
| Hosting | One small cloud VM or container service per environment; dev/test shut down when not in use |
| Tools | Open-source only: FastAPI, React, LangGraph, MLflow, GitHub Actions |
| Cost control | Cache LLM results, send only relevant pages (not whole records), per-day spending limits and alerts |
| Security testing | Free scanners (OWASP ZAP, dependency and secret scans) in CI; one external penetration test before go-live |

## 5.1b Chosen setup: under $5/month (Oracle + k3s + Argo CD)
Only cost: the OpenAI API key, capped at **$5/month**. Everything else is free.

> **Important limit:** this setup is for **public and synthetic data only** (no real patient data / PHI). Real PHI needs signed BAAs, managed secure hosting and a security test, which cost money (see 5.1a).

| Area | Choice |
|---|---|
| Server | **Oracle Cloud Always Free** Arm VM (up to 4 CPU / 24 GB RAM — check current limits) |
| Container runtime (server) | **k3s** (lightweight Kubernetes) — replaces Docker Compose on the server |
| Deployment | **Argo CD** watching Git; Helm charts in `deploy/helm/` |
| Local development | **Docker + Docker Compose** on developer laptops only |
| Image build | GitHub Actions builds Docker images (Arm64) → GitHub Container Registry (free) |
| Secrets | Sealed Secrets (encrypted secrets stored in Git) |
| Ingress + HTTPS | Traefik (built into k3s) + cert-manager with Let's Encrypt |
| Domain | Free subdomain (e.g. DuckDNS) or server IP |
| Database + vectors | PostgreSQL + pgvector running in k3s, on a persistent volume |
| File storage | Persistent volume on the server disk (encrypted) |
| Primary LLM | OpenAI `gpt-4o-mini` class for everything (incl. appeal letters); **$5 monthly limit** set in the OpenAI account |
| Fallback LLM | **Groq free tier** (Llama 3.3 70B, rate-limited) — allowed because no PHI is used |
| Embeddings | `BAAI/bge-small-en-v1.5` on CPU |
| OCR | Tesseract only (no Textract) |
| ML | XGBoost / LightGBM on CPU |
| Login + MFA | Built into FastAPI; authenticator-app MFA (free) |
| HTTPS | Self-signed or Let's Encrypt certificate (free) |
| Email | Not used; Admin resets passwords from the admin screen |
| Code, CI/CD | GitHub free plan + GitHub Actions free minutes |
| Monitoring | k3s logs, Kubernetes health checks, Uptime Kuma |
| Backups | Nightly `pg_dump` CronJob → Oracle free Object Storage |
| Security testing | Free tools only: OWASP ZAP, `pip-audit`, `npm audit`, secret scanning |
| Data | Public + synthetic (SynPUF, Synthea, CMS policies, Hugging Face) |

**Environments:** one k3s cluster, two namespaces (`dev` and `prod`), each an Argo CD application.

**Deployment flow:**
```
Laptop: docker compose up (local dev) → git push
→ GitHub Actions: tests, docker build, push image to GHCR, update image tag in Helm values
→ Argo CD (on Oracle) detects change → deploys to k3s (dev first, then prod)
```

**Keep OpenAI cost low:** at ~$0.0015 per denial, $5 covers ~2,000–3,000 denials/month. Cache results, send only relevant pages. If the limit is reached, the gateway switches to Groq automatically.

**Trade-offs:** single server (no failover); Oracle may reclaim idle free VMs — keep it in use and keep backups; Groq free tier is rate-limited.

**Moving up later:** the same Helm charts deploy to AKS (or EKS). Only Argo CD's target cluster and config values change. Budget and BAAs are needed before real PHI.

## 5.1c Phase 1 detail: data onboarding
Goal: every dataset the system needs is loaded, checked and queryable in PostgreSQL before any AI work starts.

### Sources and targets
| # | Source | What we bring in | Target (PostgreSQL) |
|---|---|---|---|
| 1 | X12 / WPC CARC + RARC lists | Denial reason codes and descriptions | `ref.carc`, `ref.rarc` |
| 2 | CMS ICD-10-CM, HCPCS/CPT descriptions | Code lists | `ref.icd10`, `ref.hcpcs` |
| 3 | CMS DE-SynPUF (sample files first) or Synthea | Patients, claims, lines | `core.patients`, `core.claims`, `core.claim_lines` |
| 4 | Label generator (`config/denial_rules.yaml`) | Denials, appeals, outcomes | `core.denials`, `core.appeals` |
| 5 | Hugging Face synthetic notes (e.g. Asclepius) | Clinical notes linked to claims | `core.documents` + files |
| 6 | CMS Medicare Coverage Database (LCD/NCD) | Policy text | `rag.policies` (chunked + embedded in Phase 3) |
| 7 | Template generator | Sample denial letters / EOBs as PDFs (clean + scan-like) | `core.documents` + files |
| 8 | Hugging Face `nielsr/funsd`, `aharley/rvl_cdip` (subset) | OCR / classifier test documents | `eval.documents` |

### Pipeline for each source
```
Download → store raw file (unchanged, with checksum) → parse → validate → load to staging
→ checks pass? → merge into core tables → log run (rows in/out, errors)
```

### Rules
- **Repeatable:** every loader can be re-run safely (idempotent, upserts by key).
- **Traceable:** every row keeps `source`, `source_file`, `loaded_at`, `run_id`.
- **Versioned:** raw files kept unchanged; reference code lists stored with an effective date.
- **Checked:** schema, required fields, valid codes, no duplicates, row counts, denial rate and reason mix match the config.
- **Size control:** start with one SynPUF sample (~100K claims) to fit the free server; add more later.
- **No PHI:** public and synthetic data only.

### Deliverables
- `src/ingest/` — one loader per source + CLI (`python -m src.ingest <source>`)
- `src/synth/` — denial label and letter generator
- `src/db/` — schemas `ref`, `core`, `rag`, `eval`, `app` + Alembic migrations
- `config/sources.yaml` (download URLs, versions) and `config/denial_rules.yaml`
- `tests/ingest/` — parser and check tests
- A data-load report: counts per table, check results

### Done when
- All 8 sources loaded into PostgreSQL on laptops (Docker Compose) and on the `dev` cluster
- All checks pass, and a full re-load gives the same counts
- A simple query can join claim → denial → notes → CARC description

## 5.2 Go-live acceptance criteria
| Area | Target (proposed) |
|---|---|
| Extraction accuracy (key fields) | ≥ 95% on the test set |
| Category classification | ≥ 90% |
| ML model AUC | ≥ 0.75 (re-measured on real outcomes after go-live) |
| Letters with valid citations | 100% |
| Reviewer acceptance of drafts | ≥ 70% with minor or no edits |
| End-to-end processing per denial | < 2 minutes |
| Access tests (user can't see others' data) | 100% pass |
| Availability | ≥ 99.5% |
| Security | No open high or critical findings |

## 5.3 Business KPIs
- Overturn rate
- Dollars recovered
- Days from denial to appeal
- Appeals per staff member per day
- Missed deadlines (target 0)
- Write-off rate

## 5.4 Risks
| Risk | Mitigation |
|---|---|
| Synthetic data doesn't match reality | Base rates on published CMS statistics; retrain on real outcomes as soon as they arrive; monitor model drift |
| LLM makes up facts | Mandatory citations, validation step, human approval |
| PHI exposure | Synthetic data only in dev/test; before go-live: OpenAI BAA / zero data retention, and a BAA with the fallback provider (Bedrock preferred; Groq for non-PHI only unless a BAA is confirmed); encryption |
| Small team (2 developers) | Managed services instead of self-hosting, phased screens, external security testing, clear owner per phase |
| OpenAI outage or rate limit | Automatic fallback to the open-source model; retries with backoff |
| Fallback model gives weaker results | Run the same evaluation set on both models; lower-confidence outputs always go to human review |
| Payer rules change | Policy index refreshed on a schedule; rules in config |
| Low user adoption | Involve reviewers early; show the reasons behind every decision |
| User sees another user's data | Server-side filtering, deny by default, row-level security, automated access tests |

## 5.5 Open decisions
- [x] Login method: separate app accounts (Admin-created, hashed passwords, lockout, required MFA)
- [x] LLM provider: OpenAI (primary), open-source model as fallback (see 3.4)
- [x] Fallback provider: AWS Bedrock or Groq (managed API, no self-hosting)
- [x] Fallback model: Llama 3.3 70B Instruct (Groq or Bedrock); confirm with the evaluation set before go-live
- [x] Hosting: Oracle Cloud Always Free + k3s + Argo CD (see 5.1b); AKS later with the same Helm charts
- [x] Fallback provider for now: Groq free tier (no PHI)
- [x] Database: PostgreSQL (with pgvector for RAG)
- [x] Dashboard: React + FastAPI
- [x] Team and owners: 2 developers own the whole project (Dev A: data/ML/AI, Dev B: API/security/dashboard)
- [x] Budget: under $5/month, OpenAI key only (see 5.1b); 5.1a applies if real PHI is added
- [ ] A few billing staff to test drafts before go-live (recommended, not required)
