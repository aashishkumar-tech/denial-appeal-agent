# Week 1 — Task Sheet

**Goal of the week:** repo works, both developers can open a pull request, database runs locally, CI is green, first reference data is loaded.

**Only this week.** Week 2 is a separate file. Don't look ahead.

---

## Naming used this week

| Thing | Pattern | Example |
|---|---|---|
| Repo | `denial-appeal-agent` | — |
| Issue ID | `P1-<nn>` | `P1-06` |
| Branch | `<type>/<area>-<what>` | `feature/data-carc-loader` |
| Commit | `<type>(<scope>): <what>` | `feat(ingest): load CARC and RARC code lists` |
| Pull request title | Same as the commit | — |

**Branch types:** `feature/`, `fix/`, `docs/`, `chore/`
**Commit types:** `feat`, `fix`, `docs`, `test`, `chore`, `refactor`
**Commit scopes this week:** `ingest`, `db`, `ci`, `deploy`, `docs`

---

# DAY 1 — Setup 🤝

> Do the repo creation **together on one screen** so both understand it.

## Dev A
| # | Task | Done when |
|---|---|---|
| 1 | Install Git, Python 3.11, Docker Desktop | `git --version`, `python --version`, `docker --version` all work |
| 2 | Configure Git identity | `git config --global user.name` / `user.email` set |
| 3 | Create GitHub account; send username to Dev B | Dev B has your handle |
| 4 | Read `docs/01-business-overview.md` and `docs/02-data-strategy.md` | You can explain what a CARC code is |
| 5 | Accept the repo invite; clone it | `git clone` works |

## Dev B
| # | Issue | Task | Done when |
|---|---|---|---|
| 1 | — | Install Git, Python 3.11, Docker Desktop | Versions print |
| 2 | — | Configure Git identity | Set |
| 3 | `P1-01` | Create private GitHub repo `denial-appeal-agent` with a README | Repo exists |
| 4 | `P1-01` | Invite Dev A as **Admin** collaborator | Invite sent |
| 5 | `P1-01` | Clone, create folder skeleton, `.gitignore`, `.github/CODEOWNERS` | Files in place |
| 6 | `P1-01` | Push this first commit **before** enabling branch protection | `git push` succeeds |
| 7 | `P1-01` | Enable branch protection on `main`: PR required, 1 approval, checks must pass | Direct push to `main` is now blocked |

**Dev B commands:**
```powershell
git clone https://github.com/<org>/denial-appeal-agent.git
cd denial-appeal-agent
mkdir docs, config, frontend, deploy, pipelines, tests
mkdir src\ingest, src\synth, src\rules, src\ml, src\llm, src\extraction, src\rag, src\agent, src\api, src\db
# create .gitignore and .github/CODEOWNERS (contents in docs/08 §Part 3)
git add .
git commit -m "chore(repo): add folder skeleton, gitignore and codeowners"
git push
```

**End of day check:** both can see the repo; folders exist; `main` is protected.

---

# DAY 2 — Practice pull request 🤝

> Today is deliberately low-risk. The point is to run the **whole PR cycle once** before real work.

## Dev A
| # | Branch | Task | Done when |
|---|---|---|---|
| 1 | `docs/add-glossary-note` | Add one line to `docs/01-business-overview.md` glossary | Committed |
| 2 | — | Push branch, open PR, request Dev B's review | PR open |
| 3 | — | Review and approve Dev B's PR | Approved |
| 4 | — | Merge own PR (squash), delete branch, pull `main` | Branch gone, `main` updated |

**Commands:**
```powershell
git checkout main
git pull
git checkout -b docs/add-glossary-note
# edit the file
git add docs/01-business-overview.md
git commit -m "docs(glossary): clarify CARC vs RARC"
git push -u origin docs/add-glossary-note
```

## Dev B
| # | Issue | Branch | Task | Done when |
|---|---|---|---|---|
| 1 | `P1-02` | `feature/infra-docker-compose` | `docker-compose.yml` with PostgreSQL + pgvector service | `docker compose up` starts the DB |
| 2 | `P1-02` | same | `.env.example` with `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `DATABASE_URL` | File committed (no real values) |
| 3 | `P1-02` | same | Add `docker compose` instructions to `README.md` | Dev A can follow them |
| 4 | — | — | Open PR, get Dev A's review, merge | Merged |

**Service name:** `denial-agent-postgres` · **Image:** `pgvector/pgvector:pg16` · **Port:** `5432`

**End of day check:** both have merged one PR; Dev A can start the database locally.

---

# DAY 3 — Database design 🤝

> Morning = design together. Afternoon = build separately.

## Morning (both, 2 hours)
Agree on paper, then write it into `docs/03-architecture-pipeline.md`:
- Schemas: `ref`, `core`, `rag`, `ml`, `eval`, `ops`
- Week 1 tables only: `ref.carc`, `ref.rarc`, `ops.load_runs`
- Conventions: snake_case; primary key `<table>_id`; every table has `source`, `source_file`, `run_id`, `created_at`

## Dev A (afternoon)
| # | Branch | Task | Done when |
|---|---|---|---|
| 1 | `docs/db-table-design` | Write column list, types, keys and constraints for `ref.carc` and `ref.rarc` | Documented |
| 2 | same | Note which fields the loaders will populate | Documented |
| 3 | — | PR → Dev B reviews | Merged |

## Dev B (afternoon)
| # | Issue | Branch | Task | Done when |
|---|---|---|---|---|
| 1 | `P1-03` | `feature/db-initial-schema` | SQLAlchemy models in `src/db/models.py` from Dev A's design | Models import cleanly |
| 2 | `P1-03` | same | `src/db/session.py`: engine and session factory | Connects to local DB |
| 3 | `P1-03` | same | Alembic init + first migration `0001_initial_schema` | `alembic upgrade head` creates the schemas and tables |
| 4 | — | — | PR → Dev A reviews | Merged |

**End of day check:** running `alembic upgrade head` on a fresh database creates `ref.carc`, `ref.rarc`, `ops.load_runs`.

---

# DAY 4 — Config + CI 🎓

> Teach-back today: **Dev B explains** what CI does and how to read a failed run.

## Dev A
| # | Issue | Branch | Task | Done when |
|---|---|---|---|---|
| 1 | `P1-05` | `feature/data-sources-config` | `config/sources.yaml`: for each source — `name`, `url`, `version`, `checksum`, `row_limit`, `target_table` | File committed |
| 2 | `P1-05` | same | `src/ingest/base.py`: `BaseLoader` with `download()`, `parse()`, `validate()`, `load()` | Class defined, typed, documented |
| 3 | `P1-05` | same | `src/ingest/downloader.py`: download with SHA-256 checksum verification, save to `data/raw/` unchanged | Re-running skips an already-valid file |
| 4 | `P1-05` | same | Unit test in `tests/ingest/test_downloader.py` | `pytest` passes |
| 5 | — | — | PR → Dev B reviews | Merged |

**Note:** `data/` is git-ignored. Never commit downloaded files.

## Dev B
| # | Issue | Branch | Task | Done when |
|---|---|---|---|---|
| 1 | `P1-04` | `feature/ci-pipeline` | `.github/workflows/ci.yml` running on every PR | Workflow appears in the Actions tab |
| 2 | `P1-04` | same | Steps: `ruff` (lint), `mypy` (types), `pytest` (tests) | All green on `main` |
| 3 | `P1-04` | same | Steps: `pip-audit` (dependency scan), secret scanning | Green |
| 4 | `P1-04` | same | Mark these checks **required** in branch protection | A failing PR cannot be merged |
| 5 | — | — | PR → Dev A reviews | Merged |

**End of day (🎓 10 min):** Dev B opens a deliberately failing run and shows Dev A how to read the log.

---

# DAY 5 — First real data 🎓

> Teach-back today: **Dev A explains** what a CARC code is and why it drives the whole product.

## Dev A
| # | Issue | Branch | Task | Done when |
|---|---|---|---|---|
| 1 | `P1-06` | `feature/data-carc-loader` | `src/ingest/carc.py`: `CarcLoader(BaseLoader)` | Loads `ref.carc` |
| 2 | `P1-06` | same | `src/ingest/rarc.py`: `RarcLoader(BaseLoader)` | Loads `ref.rarc` |
| 3 | `P1-06` | same | `src/ingest/registry.py`: maps source name → loader class | `run <source>` resolves |
| 4 | `P1-06` | same | CLI: `python -m src.ingest run carc` and `run rarc` | Commands work |
| 5 | `P1-06` | same | Idempotent upsert by code — re-running gives the same row count | Verified twice |
| 6 | `P1-06` | same | Tests in `tests/ingest/test_carc.py` | `pytest` passes |
| 7 | — | — | PR → Dev B reviews | Merged |

## Dev B
| # | Issue | Branch | Task | Done when |
|---|---|---|---|---|
| 1 | — | — | Review Dev A's loader PR carefully (first real data code) | Approved |
| 2 | `P1-06b` | `feature/db-load-run-logging` | `ops.load_runs` model + migration `0002_load_runs` | Table created |
| 3 | `P1-06b` | same | `src/db/load_run.py`: helper to start/finish a run, recording `source`, `run_id`, `rows_in`, `rows_out`, `errors`, `started_at`, `finished_at`, `status` | Loader can call it |
| 4 | `P1-06b` | same | Tests | Pass |
| 5 | — | — | PR → Dev A reviews | Merged |

**End of day (🎓 10 min):** Dev A shows a real denial reason code and explains how it decides what happens to a claim.

---

# Week 1 exit checklist

Run this together on Friday. Tick every box before starting Week 2.

- [ ] Both developers can clone, branch, commit, push and open a PR
- [ ] `main` is protected; direct pushes blocked; 1 approval required
- [ ] CODEOWNERS auto-assigns the correct reviewer
- [ ] `docker compose up` starts PostgreSQL + pgvector locally
- [ ] `alembic upgrade head` creates all schemas and Week 1 tables
- [ ] CI runs on every PR and is green on `main`
- [ ] Failing checks block a merge (tested once on purpose)
- [ ] `python -m src.ingest run carc` loads `ref.carc`
- [ ] `python -m src.ingest run rarc` loads `ref.rarc`
- [ ] Re-running both loaders gives identical row counts
- [ ] `ops.load_runs` has one row per load with status `success`
- [ ] No secrets, `.env` files or data files committed
- [ ] Both teach-back sessions happened
- [ ] Task board updated; Week 1 issues closed

---

## Issues to create on the board (Week 1)

| ID | Title | Owner |
|---|---|---|
| `P1-01` | Repo, folder skeleton, gitignore, CODEOWNERS, branch protection | Dev B |
| `P1-02` | Local Docker Compose: PostgreSQL + pgvector | Dev B |
| `P1-03` | Initial database schemas and Alembic migration | Dev B (design with Dev A) |
| `P1-04` | CI workflow: lint, types, tests, security scans | Dev B |
| `P1-05` | `sources.yaml` + base loader + checksum downloader | Dev A |
| `P1-06` | Loader: CARC and RARC code lists | Dev A |
| `P1-06b` | `ops.load_runs` logging helper | Dev B |

## If a day slips
Move the row to the next day; don't skip it. Tell the other developer at stand-up — Day 5 depends on Days 3 and 4 being finished.
