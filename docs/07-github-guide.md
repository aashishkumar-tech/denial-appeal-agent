# 7. GitHub Guide (Step by Step)

For 2 developers sharing one repository. Follow the steps in order.

---

## Part 1 — What GitHub is (in simple words)

| Word | Simple meaning |
|---|---|
| **Git** | A tool on your computer that saves the history of your code |
| **GitHub** | A website that stores your code online so both developers can share it |
| **Repository (repo)** | One project folder stored on GitHub |
| **Clone** | Copy the online project onto your computer |
| **Commit** | Save a change, with a short note about what you did |
| **Push** | Send your saved changes to GitHub |
| **Pull** | Get the other developer's changes onto your computer |
| **Branch** | Your own copy of the code where you work without disturbing anyone |
| **`main`** | The official branch. Must always work. |
| **Pull request (PR)** | "Please check my work and add it to `main`" |
| **Merge** | Adding approved work into `main` |
| **Review** | The other developer reads your code and approves it |
| **Conflict** | Both of you changed the same lines; someone must decide which version wins |

**The idea:** nobody changes `main` directly. You work on a branch, then ask the other developer to check it.

---

## Part 2 — One-time setup (both developers)

### Step 1: Install Git
Download from https://git-scm.com and install. Then check it works:
```powershell
git --version
```

### Step 2: Tell Git who you are
```powershell
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
```

### Step 3: Create a GitHub account
Sign up at https://github.com using your work email.

### Step 4: Log in from your computer
Easiest way — install GitHub CLI from https://cli.github.com, then:
```powershell
gh auth login
```
Choose: GitHub.com → HTTPS → Login with a web browser. Follow the prompts.

---

## Part 3 — Dev B creates the repo (one time only)

### Step 1: Create it on GitHub
1. Go to https://github.com → click **New repository**
2. Name: `denial-appeal-agent`
3. Visibility: **Private**
4. Tick **Add a README file**
5. Click **Create repository**

### Step 2: Add Dev A as a collaborator
Repo → **Settings** → **Collaborators** → **Add people** → type Dev A's username → role **Admin**.

Dev A then accepts the email invite.

### Step 3: Protect `main`
Repo → **Settings** → **Branches** → **Add branch protection rule**:
- Branch name pattern: `main`
- ✅ Require a pull request before merging
- ✅ Require approvals: **1**
- ✅ Require status checks to pass before merging
- Save

**What this does:** nobody can push broken code straight into `main`.

### Step 4: Copy it to your computer
```powershell
cd C:\Users\<you>\projects
git clone https://github.com/<org>/denial-appeal-agent.git
cd denial-appeal-agent
```

### Step 5: Create the folder skeleton
```powershell
mkdir docs, config, frontend, deploy, pipelines, tests
mkdir src\ingest, src\synth, src\rules, src\ml, src\llm, src\extraction, src\rag, src\agent, src\api, src\db
```

Create `.gitignore` in the root (this stops secrets and big files being uploaded):
```gitignore
.env
*.env
data/
__pycache__/
*.pyc
.venv/
node_modules/
*.log
```

Create `.github/CODEOWNERS` (auto-picks the reviewer per folder):
```
/src/ingest/      @devA
/src/synth/       @devA
/src/rules/       @devA
/src/ml/          @devA
/src/llm/         @devA
/src/extraction/  @devA
/src/rag/         @devA
/src/agent/       @devA
/pipelines/       @devA
/config/          @devA
/src/api/         @devB
/frontend/        @devB
/deploy/          @devB
/.github/         @devB
/src/db/          @devA @devB
/docs/            @devA @devB
```
Replace `@devA` and `@devB` with the real GitHub usernames.

### Step 6: Send it to GitHub
```powershell
git add .
git commit -m "chore: add folder structure, gitignore and codeowners"
git push
```

> Note: `main` is now protected, so this direct push may be rejected. If it is, do this first commit **before** turning on protection in Step 3, or use a branch and pull request as in Part 5.

---

## Part 4 — Dev A joins (one time only)

### Step 1: Accept the invite
Check your email or https://github.com/notifications and accept.

### Step 2: Copy the project to your computer
```powershell
cd C:\Users\<you>\projects
git clone https://github.com/<org>/denial-appeal-agent.git
cd denial-appeal-agent
```

Done. You now have the same files as Dev B.

---

## Part 5 — The daily cycle (both developers, every task)

This is the part you repeat forever. Six steps.

### Step 1: Get the latest code
```powershell
git checkout main
git pull
```
**Why:** so you start from the other developer's newest work.

### Step 2: Create your branch
```powershell
git checkout -b feature/carc-loader
```
**Why:** your own space. `main` stays safe.

Naming: `feature/<what-you-build>`, `fix/<what-you-fix>`, `docs/<what-you-write>`

### Step 3: Do the work, then save it
```powershell
git status                    # see what you changed
git add src/ingest/carc.py    # pick the files to save
git commit -m "feat(ingest): load CARC and RARC code lists"
```
**Tip:** commit small and often. Several small commits beat one huge one.

### Step 4: Send your branch to GitHub
```powershell
git push -u origin feature/carc-loader
```
(After the first push on that branch, just `git push`.)

### Step 5: Open a pull request
1. Go to the repo on GitHub — a yellow **Compare & pull request** button appears
2. Click it
3. Title: what you did
4. Description: why, how to test it
5. Click **Create pull request**

The other developer is added as reviewer automatically (CODEOWNERS).

### Step 6: Get reviewed, then merge
1. The other developer reads it and clicks **Approve** (or asks for changes)
2. If changes are asked for: fix them on your computer, then `git add`, `git commit`, `git push` — the pull request updates itself
3. Once approved: click **Squash and merge**, then **Delete branch**

### Step 7: Clean up on your computer
```powershell
git checkout main
git pull
git branch -d feature/carc-loader
```

**That's the whole cycle.** Repeat for every task.

---

## Part 6 — Staying in sync while you work

If your branch takes more than a day, update it daily so you don't fall behind:
```powershell
git fetch origin
git rebase origin/main
```
**What this does:** puts the other developer's newest work underneath yours.

If Git reports a conflict, see Part 7.

---

## Part 7 — Conflicts (when both changed the same lines)

Git will say `CONFLICT` and name the file. Don't panic.

### Step 1: Open the file
You'll see:
```
<<<<<<< HEAD
their version of the line
=======
your version of the line
>>>>>>> your-branch
```

### Step 2: Decide what's correct
Delete the `<<<<<<<`, `=======` and `>>>>>>>` lines, and leave the code you want (sometimes both versions, sometimes one).

### Step 3: Finish
```powershell
git add <the-file>
git rebase --continue
git push --force-with-lease
```

### How to avoid conflicts
- Stay in **your own folders** (see CODEOWNERS)
- Keep branches **short** (1–3 days)
- **Rebase daily**
- Tell the other developer in the daily check-in before editing shared files: `src/db/`, `docs/`, `docker-compose.yml`, `requirements.txt`

---

## Part 8 — The task sheet (GitHub Projects, free)

### Step 1: Create the board
Repo → **Projects** tab → **New project** → **Board** → name it `Denial Agent`.

### Step 2: Create the columns
`Todo` → `In progress` → `In review` → `Done`

### Step 3: Add tasks as issues
Repo → **Issues** → **New issue**. One issue per task, for example:
- Title: `P1-06 Loader: CARC + RARC code lists`
- Assignee: Dev A
- Add it to the project board

Use the current week's task sheet (e.g. `docs/week-01-tasks.md`) to create that week's issues.

### Step 4: Link work to tasks
In your pull request description, write `Closes #6`. When the pull request merges, the issue closes and moves to `Done` automatically.

---

## Part 9 — Everyday commands (cheat sheet)

```powershell
git status                     # what have I changed?
git pull                       # get the latest code
git checkout -b feature/x      # start a new branch
git add .                      # stage all my changes
git commit -m "message"        # save with a note
git push                       # send to GitHub
git checkout main              # switch back to main
git log --oneline -10          # last 10 changes
git diff                       # show my unsaved changes
git restore <file>             # undo changes to a file
```

---

## Part 10 — Rules to never break

1. **Never push straight to `main`** — always use a branch and pull request
2. **Never commit secrets** — no `.env`, API keys or passwords
3. **Never commit large data files** — `data/` is ignored; loaders download the data
4. **Never merge a pull request with failing checks**
5. **Always pull before starting** a new branch
6. **Always review within one working day** so the other person isn't blocked

---

## Part 11 — First week checklist

| # | Who | Task |
|---|---|---|
| 1 | Both | Install Git, set name and email, create GitHub account |
| 2 | Dev B | Create the private repo |
| 3 | Dev B | Add Dev A as Admin collaborator |
| 4 | Dev B | Push folder skeleton, `.gitignore`, CODEOWNERS |
| 5 | Dev B | Turn on branch protection for `main` |
| 6 | Dev A | Accept invite and clone the repo |
| 7 | Both | Create the project board and the first 16 issues |
| 8 | Both | Each do one small practice pull request to test the flow |
