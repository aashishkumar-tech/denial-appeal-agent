# 10. Shared Workspace, Different Machines

## The setup

| Machine | Purpose | What happens there |
|---|---|---|
| **This workspace** (shared, turn by turn) | **Code generation only** | Write code with Copilot, commit, push. |
| **Dev A's device** | Run & test | venv, Docker, database, pipelines, ML training. |
| **Dev B's device** | Run & test | venv, Docker, database, API, frontend, k3s. |

**Nothing is ever run, installed or tested in this workspace.** No `.venv`, no `docker compose up`, no `pytest`, no data downloads. It is a text editor with Copilot attached.

Two risks follow, and these rules prevent both:
1. **Overwriting each other** — two people editing the same checkout at different times.
2. **"Works on my machine"** — code generated here was never executed, so it must be correct by construction.

---

## 10.1 Golden rules

| # | Rule |
|---|---|
| 1 | **Pull before you generate.** `git checkout main && git pull` — never generate code on top of a stale checkout. |
| 2 | **Push before you hand over.** Never leave uncommitted work in the shared workspace. |
| 3 | **One branch at a time** in the shared workspace. Say in the stand-up when you take it. |
| 4 | **Never run anything here.** Generated code is verified on your own device, and by CI. |
| 5 | **Nothing machine-specific in Git.** No absolute paths, no `.env`, no local file locations. |
| 6 | **Code is unverified until CI is green.** Pushing from here proves nothing — open the pull request and wait for the checks. |

---

## 10.2 The turn-by-turn cycle

```
  shared workspace                 your own device
  ----------------                 ---------------
  1. git pull
  2. git checkout -b feature/x
  3. generate code with Copilot
  4. commit + push          ---->  5. git pull
                                   6. run: pytest, docker compose, pipeline
                                   7. found a bug?
                                        small  -> fix + push from your device
                                        large  -> take the workspace again
  8. open pull request
  9. CI green + review      ---->  10. merge
```

Fixing small bugs directly on your own device is fine and encouraged — just push from there. The shared workspace is not the only place you may commit from; it is only the place you **generate**.

---

## 10.3 Handover checklist

**Before you leave the workspace:**
```powershell
git status              # must show a clean tree
git add .
git commit -m "feat(ingest): ..."
git push
```
- [ ] `git status` is clean (nothing uncommitted, nothing untracked that matters)
- [ ] Branch pushed to GitHub
- [ ] Task board updated
- [ ] Told the other developer the workspace is free

**When you take the workspace:**
```powershell
git status              # confirm it's clean — if not, STOP and ask
git checkout main
git pull
git checkout -b feature/<your-task>
```

> If you find uncommitted changes that aren't yours: **do not delete them.** Ask the other developer first.

---

## 10.4 Because code is written blind here

Nothing is executed in this workspace, so these habits are not optional:

| Habit | Why |
|---|---|
| Write the test alongside the code | The test is the specification; your device proves it. |
| Keep changes small | An unverified 500-line change is a bad bet. |
| Never invent APIs or library functions | You cannot run it to find out. Read the real source first. |
| Pin every new dependency in `requirements.txt` | The other device must be able to reproduce it. |
| Add every new setting to `.env.example` | Otherwise the other device fails on startup. |
| Let CI be the gate | Lint, type-check and tests run on push — that is your real feedback loop. |

---

## 10.5 Making code machine-independent

The shared workspace runs Windows. The two run-machines may be Windows, macOS or Linux. Code must work on all of them.

| Problem | Rule |
|---|---|
| Paths | Use `pathlib.Path`, never `"C:\\Users\\..."` or hard-coded `/` |
| Project root | Resolve from the file: `Path(__file__).resolve().parents[2]` |
| Data locations | Read from `.env` (`DATA_RAW_DIR`, `DOCUMENT_STORE_DIR`), never hard-coded |
| Line endings | Handled by `.gitattributes` (LF everywhere except `.ps1`/`.bat`) |
| Python version | Python 3.11 on both machines |
| Dependencies | Pin exact versions in `requirements.txt`; both run `pip install -r requirements.txt` |
| Database | Always through Docker Compose, never a locally installed PostgreSQL |
| Secrets | Each machine has its **own** `.env`, copied from `.env.example` |
| Commands | Use `python -m src.ingest run carc`, not a path to a script |

**Good:**
```python
from pathlib import Path
import os

PROJECT_ROOT = Path(__file__).resolve().parents[2]
RAW_DIR = Path(os.getenv("DATA_RAW_DIR", PROJECT_ROOT / "data" / "raw"))
```

**Bad:**
```python
RAW_DIR = "C:\\Users\\dev\\projects\\denial-appeal-agent\\data\\raw"
```

---

## 10.6 Each run-machine's one-time setup

Do this on **your own device only** — never in the shared workspace:

```powershell
git clone https://github.com/<org>/denial-appeal-agent.git
cd denial-appeal-agent

python -m venv .venv
.\.venv\Scripts\Activate.ps1          # macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt

Copy-Item .env.example .env           # then fill in your own values
docker compose up -d                  # starts PostgreSQL + pgvector
alembic upgrade head                  # creates the schemas
pytest                                # everything should pass
```

**Rule:** if CI is green but it fails on your machine, the code is machine-dependent. Fix the code, not your machine.

---

## 10.7 Where each thing lives

| Item | Shared workspace | Your own device | Git |
|---|---|---|---|
| Source code | ✅ generated here | ✅ run here | ✅ |
| `.env` | ❌ not needed | ✅ own copy | ❌ never |
| `.venv` | ❌ not needed | ✅ | ❌ |
| `data/` downloads | ❌ never | ✅ | ❌ never |
| Docker volumes / database | ❌ never | ✅ separate per device | ❌ |
| Trained models | ❌ never | ✅ `python -m src.ml train` | ❌ |

**Only Git is shared.** Everything else is rebuilt from the repo on each run-machine.

---

## 10.8 Definition of done (adds to the normal checklist)
- [ ] No absolute paths anywhere in the change
- [ ] New config values added to `.env.example`
- [ ] New dependencies pinned in `requirements.txt`
- [ ] Actually executed on at least one run-machine (not just generated)
- [ ] CI green on the pull request
- [ ] Shared workspace left clean and pushed
