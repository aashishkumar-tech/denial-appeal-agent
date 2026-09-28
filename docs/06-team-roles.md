# 6. Team Roles and Responsibilities

Two developers own the whole project: build, testing, deployment, support and technical decisions.

| | Dev A | Dev B |
|---|---|---|
| **Title** | Data & AI Engineer | Platform & Application Engineer |
| **Main language** | Python | Python (FastAPI) + TypeScript (React) |
| **Owns** | Data, ML, LLM gateway, RAG, agent | Infrastructure, API, security, dashboard |
| **Backup for** | Platform deployments, API reviews | Data loaders, ML pipeline runs |

---

## 6.1 Dev A: Data & AI Engineer

### Responsibilities
- Bring all data onto the platform and keep it correct
- Build the denial label and letter generators
- Build the rules engine and ML scoring
- Build the LLM gateway (OpenAI + Groq fallback), prompts and JSON schemas
- Build OCR, document classification and field extraction
- Build the policy index (RAG) and the LangGraph agent
- Evaluate models and keep the test sets
- Watch OpenAI spend and model quality

### Folders owned
`src/ingest/`, `src/synth/`, `src/rules/`, `src/ml/`, `src/llm/`, `src/extraction/`, `src/rag/`, `src/agent/`, `pipelines/`, `config/denial_rules.yaml`, `config/sources.yaml`, `config/llm.yaml`, `tests/` for these folders

### Work by phase
| Phase | Weeks | Dev A tasks |
|---|---|---|
| 1. Data onboarding | 1–4 | Loaders for CARC/RARC, ICD-10, HCPCS, SynPUF/Synthea, notes, LCD/NCD, test documents; denial label generator; letter/EOB PDF generator; data checks; load report |
| 1b. Platform foundation | 3–5 | Review Helm values for data jobs; run loaders on `dev` cluster |
| 2. Rules + ML | 6–9 | Rules engine; feature building; overturn model (XGBoost); priority score; MLflow tracking; model report |
| 3. Documents + RAG + Agent | 9–15 | Tesseract OCR; document classifier; LLM extraction to JSON; LLM gateway with fallback, caching and cost limits; policy chunking + embeddings in pgvector; agent tools and graph; letter templates; citation checks; evaluation set results |
| 4. API + Dashboard | support | Python functions the API calls (score, run agent, extract); explain-why data for the case screen |
| 5. Hardening | 16–19 | Model and extraction accuracy checks; prompt safety tests; load tests for agent runs |
| 6. Go-live | 20–21 | Final evaluation report; runbook for data loads and retraining |
| 7. Improve | ongoing | Retrain on new outcomes; improve templates from reviewer edits; add payers and policies |

### Deliverables
- All data in PostgreSQL with checks passing
- Trained, versioned model with a report
- Working agent that drafts cited appeal letters
- Evaluation results for extraction, classification, model and letters

---

## 6.2 Dev B: Platform & Application Engineer

### Responsibilities
- Set up and run the platform: Oracle VM, k3s, Argo CD, CI/CD
- Own the database setup, migrations, backups and restore
- Build the FastAPI back end and all endpoints
- Build login, MFA, roles, per-user data filtering and row-level security
- Build the React dashboard, document viewer and side panel
- Own security: secrets, HTTPS, scans, audit log
- Monitoring, alerts and uptime

### Folders owned
`src/api/`, `src/db/` (shared with Dev A for schemas), `frontend/`, `deploy/helm/`, `deploy/argocd/`, `.github/workflows/`, `docker-compose.yml`, Dockerfiles, `tests/` for these folders

### Work by phase
| Phase | Weeks | Dev B tasks |
|---|---|---|
| 1. Data onboarding | 1–4 | Repo setup; `docker-compose.yml` (PostgreSQL + pgvector); database schemas and Alembic migrations with Dev A; document file storage; CI with tests and lint |
| 1b. Platform foundation | 3–5 | Oracle VM; k3s; Argo CD; Helm chart; GHCR image builds (Arm64); Sealed Secrets; Traefik + cert-manager; `dev` and `prod` namespaces; backup CronJob; Uptime Kuma |
| 4. API + Dashboard | 3–16 | FastAPI endpoints; login with hashed passwords, lockout, MFA and JWT; roles and permissions; row-level security; audit log; document upload and page images; React screens: queue, case detail, appeal review, tracking, my items, admin, KPIs; PDF.js viewer and side panel |
| 5. Hardening | 16–19 | Access tests (user A can't see user B's data); OWASP ZAP, dependency and secret scans; load tests; backup restore test |
| 6. Go-live | 20–21 | Production release through Argo CD; create Admin account; user guide; support runbook |
| 7. Improve | ongoing | Analytics screen; UI improvements; platform updates and patches |

### Deliverables
- Running `dev` and `prod` environments deployed by Argo CD
- Secure API with login, MFA, roles and audit log
- Full dashboard with document viewer and side panel
- Passing security and access tests; tested backups

---

## 6.3 Shared work
| Area | How it's shared |
|---|---|
| Database schemas | Dev A designs data tables; Dev B owns migrations, security and performance |
| API contract | Agree endpoint inputs/outputs before building (OpenAPI spec) |
| Code review | Every pull request reviewed by the other developer |
| Testing | Each writes tests for their own code; both run the end-to-end test before release |
| Releases | Dev B deploys; Dev A confirms data and model checks pass |
| Support | Weekly rotation for first response; issue goes to the owner of that area |
| Documentation | Each updates the docs for their area in the same pull request |

## 6.4 Handoff points
| When | From → To | What |
|---|---|---|
| Week 2 | Dev B → Dev A | Local database running in Docker Compose |
| Week 4 | Dev A → Dev B | All data loaded; table list for the API |
| Week 5 | Dev B → Dev A | `dev` cluster ready for data jobs |
| Week 9 | Dev A → Dev B | Scoring functions ready for the queue screen |
| Week 15 | Dev A → Dev B | Agent and extraction functions ready for the case and review screens |
| Week 16 | Both | Feature freeze; hardening starts |

## 6.5 Working rhythm
- **Daily:** 15-minute check-in: done, next, blocked
- **Weekly:** plan the week; demo what works; check OpenAI spend and server health
- **Every pull request:** tests pass in CI, reviewed by the other developer, docs updated
- **Branches:** `main` deploys to `prod`; feature branches merge to `main` through pull requests; `dev` deploys from the latest build

## 6.6 Decision rules
- Each developer decides within their own area
- Cross-area decisions (schemas, API contract, new tools, anything with cost) are agreed by both and noted in the docs
- Anything that would add cost above the $5/month budget needs both to agree
