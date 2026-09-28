# 3. Architecture & Pipeline

## 3.1 High-level flow
```
┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
│ Data Layer │──►│ Processing │──►│ Decision   │──►│ Agent      │
│ claims,835,│   │ OCR, LLM   │   │ Rules + ML │   │ RAG, draft │
│ docs, policy│  │ extraction │   │ scoring    │   │ letter     │
└────────────┘   └────────────┘   └────────────┘   └─────┬──────┘
                                                         ▼
┌────────────┐   ┌────────────┐   ┌──────────────────────────────┐
│ Retraining │◄──│ Tracking   │◄──│ Human Review (Dashboard)     │
│ (ML loop)  │   │ outcomes   │   │ approve / edit / reject      │
└────────────┘   └────────────┘   └──────────────────────────────┘
```

## 3.2 Pipeline stages
| # | Stage | Input | Output | Tech |
|---|---|---|---|---|
| 1 | Ingest | Raw files (CSV, 835, PDF) and user uploads from the dashboard | Raw tables and stored files | Python, pandas; encrypted file storage |
| 2 | Label generation (dev/test and initial model training only) | Claims | Denials, appeals | Python + YAML rules |
| 3 | Document extraction | PDF or scan | JSON fields | OCR + LLM with JSON schema |
| 4 | Normalize & classify | CARC/RARC/group | Category | Mapping table + LLM fallback |
| 5 | Rules engine | Denial | Route or stop | Python |
| 6 | ML scoring | Features | P(overturn), priority | XGBoost / LightGBM |
| 7 | Policy index | LCD/NCD docs | Vector index | Embeddings + vector DB |
| 8 | Agent | Denial + context | Evidence, draft letter | LangGraph |
| 9 | Review | Draft | Approved letter | Dashboard |
| 10 | Track & learn | Outcomes | New training data | Scheduled job |

## 3.3 Agent design
**Tools:**
| Tool | Purpose |
|---|---|
| `get_claim(claim_id)` | Claim and denial details |
| `get_documents(claim_id)` | Clinical notes |
| `search_policy(cpt, icd10, query)` | RAG over payer policies |
| `check_authorization(claim_id)` | Look for an existing authorization |
| `get_history(payer, carc)` | Past win rate |
| `draft_letter(context, template)` | Generate the letter with citations |
| `create_task(type, details)` | Documentation query or review task |

**Graph:**
```
start → load_context → search_policy → match_criteria
      → [gap?] → create_doc_query → end
      → [ok]   → draft_letter → validate_citations → send_to_review → end
```

**Validation before review:**
- Every statement has a citation (doc_id and snippet)
- Codes, dates and amounts match the source data
- Uses the correct template for the category

## 3.4 Tech stack
| Layer | Choice |
|---|---|
| Language | Python 3.11+ |
| Storage | **PostgreSQL** (decided) |
| Vector DB | **pgvector** extension in the same PostgreSQL |
| LLM (primary) | **OpenAI API** (`gpt-4o-mini` class, $5/month cap) |
| LLM (fallback) | **Llama 3.3 70B Instruct** via Groq free tier now (no PHI); AWS Bedrock / Azure option later for PHI |
| Embeddings | Open-source `BAAI/bge-small-en-v1.5` on CPU (free, one model for the whole index) |
| OCR | Tesseract (free) |
| ML | scikit-learn, XGBoost, MLflow |
| Agent | LangGraph |
| API | FastAPI |
| Dashboard | React (TypeScript) + PDF.js |
| Scheduled jobs | Kubernetes CronJobs (backups, retraining, follow-ups) |
| Server | Oracle Cloud Always Free Arm VM |
| Runtime | **k3s** on the server; Docker Compose on laptops only |
| Deployment | **Argo CD** (GitOps) + Helm charts; images built by GitHub Actions, stored in GHCR |
| Secrets | Sealed Secrets |
| Ingress / HTTPS | Traefik (k3s) + cert-manager (Let's Encrypt) |

### LLM routing and fallback
```
Request → LLM Gateway (src/llm/) → OpenAI
                                   │ fails (timeout, 429 rate limit, 5xx, outage)
                                   ▼ after retries
                               Open-source model (AWS Bedrock or Groq API)
```
- All code calls one internal interface (`src/llm/`), never a provider directly, so providers can be swapped by config.
- Fallback triggers: timeout, rate limit, server error, or the output fails JSON-schema validation twice.
- The same prompts, JSON schemas and validation run on both models.
- Every call logs provider, model, latency, tokens and whether fallback was used (shown on the admin screen).
- Settings in `config/llm.yaml`: primary model, fallback model, timeouts, retries.
- **Embeddings:** vectors from different models can't be mixed. Pick one embedding model for the policy index; if switching, re-index. Keep the embedding model name stored with each vector.
- **Why managed APIs:** with a 2-developer team, running GPU servers is too much operational work. Bedrock and Groq are called like OpenAI, so the gateway stays simple.
- **PHI routing (before real data):** every provider that receives PHI needs a signed BAA. Use OpenAI only with a BAA / zero data retention. AWS Bedrock is the preferred fallback for PHI (AWS offers a BAA — confirm Bedrock is covered for your account). Use Groq only for non-PHI requests unless Groq confirms a BAA in writing.

### PostgreSQL layout
| Schema | Contents |
|---|---|
| `ref` | CARC, RARC, ICD-10, HCPCS/CPT code lists |
| `core` | patients, claims, claim_lines, denials, appeals, documents |
| `eval` | test documents and labels for OCR / extraction evaluation |
| `rag` | policies, policy_chunks (with `vector` column via pgvector) |
| `ml` | features, scores, model_versions |
| `app` | users, teams, assignments, audit_log |

- Row-level security on case tables using `assigned_user_id` and `team_id`.
- Migrations with Alembic; runs locally via Docker Compose and on the server as a k3s StatefulSet (`pgvector/pgvector` image, persistent volume).

## 3.5 Planned folder layout
```
IDP/
├── README.md
├── docs/
├── config/            # denial_rules.yaml, thresholds.yaml, settings
├── data/              # raw/, processed/ (git-ignored)
├── src/
│   ├── ingest/        # download and load datasets
│   ├── synth/         # denial label generator
│   ├── llm/           # LLM gateway: OpenAI primary, open-source fallback
│   ├── db/            # PostgreSQL models, Alembic migrations
│   ├── extraction/    # OCR + LLM extraction
│   ├── rules/         # rules engine
│   ├── ml/            # features, train, score
│   ├── rag/           # policy indexing and search
│   ├── agent/         # LangGraph agent and tools
│   ├── api/           # FastAPI service
├── frontend/          # React dashboard
├── pipelines/         # batch jobs (run as k3s CronJobs)
├── tests/
├── deploy/
│   ├── helm/          # Helm chart: api, frontend, worker, postgres, mlflow
│   │   ├── values-dev.yaml
│   │   └── values-prod.yaml
│   └── argocd/        # Argo CD Application definitions (dev, prod)
├── .github/workflows/ # CI: tests, scans, build + push images
└── docker-compose.yml # local development only
```

## 3.6 API endpoints (draft)
| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/denials` | List with filters |
| GET | `/denials/{id}` | Detail + score + evidence |
| POST | `/denials/{id}/run-agent` | Generate an appeal draft |
| POST | `/appeals/{id}/approve` | Approve and submit |
| POST | `/appeals/{id}/reject` | Reject with a reason |
| POST | `/appeals/{id}/outcome` | Record the result |
| GET | `/metrics` | KPIs for the dashboard |
| POST | `/documents/upload` | Upload one or more files, optionally linked to a case |
| GET | `/documents/{id}` | Metadata, status, extracted fields |
| GET | `/documents/{id}/pages/{n}` | Page image via short-lived link |
| PATCH | `/documents/{id}/fields` | Save user corrections |
| GET | `/cases/{id}/documents` | All documents on a case (for timeline and side panel) |

## 3.7 Security & compliance
- Role-based access (see Dashboard roles)
- Audit log of every agent action and user decision
- Secrets in environment variables or a key vault, never in code
- PHI encrypted at rest and in transit
- Separate environments: dev, test/staging, production; real PHI only in production
- CI/CD with automated tests, security scans and approvals before deploy
- Monitoring and alerts: API errors, latency, LLM fallback rate, model drift, failed jobs
- Backups and disaster recovery for PostgreSQL and file storage
- LLM provider must meet HIPAA requirements (BAA) before real data is used
