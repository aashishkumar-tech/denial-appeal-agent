# 8. Dev A — Work Plan (Data & AI Engineer)

**Role title:** Data & AI Engineer
**Owns:** data platform, ML scoring, LLM gateway, document intelligence, RAG, agent
**Primary language:** Python 3.11
**GitHub handle placeholder:** `@devA`

> All names below (modules, tables, configs, branches, issues) are the agreed conventions. Use them exactly so the codebase stays consistent.

---

## 9.1 Scope summary

| Area | Included | Not included (Dev B) |
|---|---|---|
| Data ingestion | Loaders, validation, load reports | File storage service, migrations |
| Synthetic data | Denial labels, letter/EOB PDFs | — |
| Rules & ML | Rules engine, features, models, MLflow | Serving endpoints |
| LLM | Gateway, prompts, schemas, cost control | API auth |
| Documents | OCR, classification, extraction | Upload API, viewer UI |
| RAG | Chunking, embeddings, retrieval | — |
| Agent | LangGraph graph, tools, letter drafting | Review UI |

---

## 9.2 Naming conventions (Dev A)

### Python modules
| Path | Purpose |
|---|---|
| `src/ingest/` | One module per source: `carc.py`, `rarc.py`, `icd10.py`, `hcpcs.py`, `synpuf.py`, `synthea.py`, `clinical_notes.py`, `cms_policies.py`, `eval_docs.py` |
| `src/ingest/base.py` | `BaseLoader` class: `download()`, `parse()`, `validate()`, `load()` |
| `src/ingest/registry.py` | Maps source name → loader class |
| `src/synth/` | `denial_generator.py`, `appeal_outcome_generator.py`, `letter_renderer.py`, `scan_effects.py` |
| `src/rules/` | `engine.py`, `carc_mapping.py`, `deadlines.py`, `routing.py` |
| `src/ml/` | `features.py`, `train_overturn.py`, `score.py`, `priority.py`, `registry.py` |
| `src/llm/` | `gateway.py`, `providers/openai_provider.py`, `providers/groq_provider.py`, `schemas.py`, `prompts/`, `cache.py`, `usage_tracker.py` |
| `src/extraction/` | `ocr.py`, `classifier.py`, `field_extractor.py`, `confidence.py` |
| `src/rag/` | `chunker.py`, `embedder.py`, `indexer.py`, `retriever.py` |
| `src/agent/` | `graph.py`, `tools.py`, `criteria_matcher.py`, `letter_builder.py`, `citation_validator.py` |
| `pipelines/` | `load_all.py`, `generate_denials.py`, `build_policy_index.py`, `train_models.py`, `score_backlog.py` |

### Config files (`config/`)
| File | Contents |
|---|---|
| `sources.yaml` | Download URLs, versions, checksums, row limits |
| `denial_rules.yaml` | Denial rates, CARC mix, outcome probabilities |
| `carc_categories.yaml` | CARC/RARC → root-cause category mapping |
| `thresholds.yaml` | Probability cut-offs, minimum dollar amount, deadline alert days |
| `llm.yaml` | Primary/fallback model, timeouts, retries, monthly cost cap |
| `payer_policies.yaml` | Payer appeal windows and levels |

### Database objects (Dev A proposes, Dev B migrates)
| Schema | Tables |
|---|---|
| `ref` | `carc`, `rarc`, `icd10`, `hcpcs`, `payers` |
| `core` | `patients`, `claims`, `claim_lines`, `denials`, `appeals`, `documents` |
| `rag` | `policies`, `policy_chunks` |
| `ml` | `features`, `scores`, `model_versions` |
| `eval` | `documents`, `labels`, `runs` |
| `ops` | `load_runs`, `llm_usage` |

**Conventions:** snake_case; singular column names; primary keys `<table>_id`; every table has `source`, `source_file`, `run_id`, `created_at`.

### Branch names
`feature/data-<source>-loader`, `feature/ml-<model>`, `feature/llm-<capability>`, `feature/agent-<node>`, `fix/<area>-<issue>`

### Commit scopes
`ingest`, `synth`, `rules`, `ml`, `llm`, `extraction`, `rag`, `agent`, `pipelines`

### CLI commands
```bash
python -m src.ingest run <source>        # load one source
python -m src.ingest run-all             # load everything
python -m src.synth denials              # generate denial labels
python -m src.synth letters              # render denial letter PDFs
python -m src.ml train                   # train overturn model
python -m src.rag index                  # build policy index
python -m src.agent run --claim-id <id>  # run the agent on one claim
```

---

## 9.3 Task backlog (issue IDs)

### Phase 1 — Data onboarding (Weeks 1–4)
| ID | Task | Est. | Depends on |
|---|---|---|---|
| `P1-05` | `config/sources.yaml` + download helper with checksum verification | 1d | P1-01 |
| `P1-06` | Loader: CARC + RARC (`src/ingest/carc.py`, `rarc.py`) | 1d | P1-03, P1-05 |
| `P1-07` | Loader: ICD-10 + HCPCS (`icd10.py`, `hcpcs.py`) | 1d | P1-03 |
| `P1-08` | Loader: SynPUF/Synthea claims, ~100K sample (`synpuf.py`) | 3d | P1-03 |
| `P1-09` | Denial label generator + `denial_rules.yaml` | 3d | P1-08 |
| `P1-10` | Loader: clinical notes linked to claims (`clinical_notes.py`) | 2d | P1-08 |
| `P1-11` | Loader: CMS LCD/NCD policies (`cms_policies.py`) | 2d | P1-03 |
| `P1-12` | Denial letter / EOB PDF generator + scan effects | 3d | P1-09 |
| `P1-13` | Loader: OCR/classifier evaluation documents (`eval_docs.py`) | 1d | P1-03 |
| `P1-15` | Data quality checks + load report (`ops.load_runs`) | 2d | P1-06…P1-13 |
| `P1-16` | End-to-end query test: claim → denial → notes → CARC text | 1d | P1-15 |

**Phase 1 exit:** all sources loaded on laptop and `dev`; all checks pass; full re-load reproduces identical counts.

### Phase 2 — Rules + ML (Weeks 6–9)
| ID | Task | Est. |
|---|---|---|
| `P2-01` | `carc_categories.yaml` + category mapper | 2d |
| `P2-02` | Rules engine: PR routing, deadline, minimum amount, appeal levels | 3d |
| `P2-03` | Payer deadline calculator (`deadlines.py`, `payer_policies.yaml`) | 2d |
| `P2-04` | Feature builder (`features.py`) | 3d |
| `P2-05` | Overturn model training + MLflow tracking | 3d |
| `P2-06` | Priority score (expected recovery ÷ effort, deadline boost) | 2d |
| `P2-07` | Batch scoring pipeline (`score_backlog.py`) | 2d |
| `P2-08` | Model report: AUC, calibration, feature importance | 2d |

**Phase 2 exit:** AUC ≥ 0.75 on held-out data; every denial in `ml.scores` with a priority and an explanation.

### Phase 3 — Documents + RAG + Agent (Weeks 9–15)
| ID | Task | Est. |
|---|---|---|
| `P3-01` | LLM gateway with OpenAI primary + Groq fallback | 3d |
| `P3-02` | Cost cap, caching, usage tracking (`ops.llm_usage`) | 2d |
| `P3-03` | JSON schemas + validation for every LLM output | 2d |
| `P3-04` | OCR service (Tesseract) with per-page confidence | 3d |
| `P3-05` | Document classifier (15 classes) | 3d |
| `P3-06` | Field extractor for denial letters / EOBs | 4d |
| `P3-07` | Policy chunking + embeddings + pgvector index | 3d |
| `P3-08` | Retriever with CPT/ICD filtering | 2d |
| `P3-09` | Criteria matcher: policy criteria vs chart evidence | 4d |
| `P3-10` | Agent graph (LangGraph) + tools | 5d |
| `P3-11` | Letter templates per denial category | 3d |
| `P3-12` | Citation validator (no uncited claims) | 2d |
| `P3-13` | Evaluation harness + results report | 3d |

**Phase 3 exit:** extraction ≥ 95%, classification ≥ 90%, 100% of drafts pass citation validation.

### Phase 5–7
| ID | Task |
|---|---|
| `P5-01` | Model + extraction accuracy regression suite |
| `P5-02` | Prompt-injection and safety tests |
| `P5-03` | Load test: 100 agent runs |
| `P6-01` | Final evaluation report |
| `P6-02` | Runbook: data loads, retraining, index refresh |
| `P7-xx` | Retraining on real outcomes; template tuning from reviewer edits |

---

## 9.4 Interfaces Dev A provides to Dev B

These are the contracts Dev B builds the API on. Agree signatures **before** Phase 4.

```python
# src/ml/score.py
def score_denial(denial_id: str) -> ScoreResult
# → probability, expected_recovery, priority, top_factors[], rule_decisions[]

# src/rules/engine.py
def route_denial(denial_id: str) -> RoutingDecision
# → action ('appeal'|'corrected_claim'|'patient_billing'|'write_off'|'human_review'), reason

# src/agent/graph.py
def run_agent(claim_id: str, user_id: str) -> AgentResult
# → status, draft_letter, citations[], evidence_gaps[], trace_id

# src/extraction/field_extractor.py
def extract_document(document_id: str) -> ExtractionResult
# → fields{}, confidences{}, page_boxes[], doc_class

# src/rag/retriever.py
def search_policy(cpt: str, icd10: str, query: str) -> list[PolicyChunk]
```

**Rule:** these functions never handle authentication or user filtering — Dev B enforces that in the API layer.

---

## 9.5 Definition of done (every Dev A task)
- [ ] Unit tests in `tests/<area>/` passing
- [ ] Type hints and docstrings on public functions
- [ ] Config values in YAML, never hard-coded
- [ ] Logs include `run_id` / `trace_id`
- [ ] No PHI, secrets or large data files committed
- [ ] Docs updated in the same pull request
- [ ] Reviewed and approved by Dev B

---

## 9.6 Quality gates Dev A owns
| Metric | Target |
|---|---|
| Field extraction accuracy | ≥ 95% |
| Document classification | ≥ 90% |
| Overturn model AUC | ≥ 0.75 |
| Drafts with valid citations | 100% |
| Agent run time per denial | < 2 min |
| OpenAI monthly spend | ≤ $5 |
