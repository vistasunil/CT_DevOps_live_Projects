# Project 03 — Team Branching Strategy & Release Tagging Workflow

| | |
|---|---|
| **Module** | 2 — Git |
| **Level** | L1 — Associate |
| **Role you play** | DevOps Engineer owning source control standards for a 6-developer team |
| **Scope** | `Common-Git-Commands.md`, `Git-Merge.md` — clone, branch, commit, merge, tags, remotes, `.gitignore` |

---

## 1. Business problem

A six-developer team commits directly to `main`. Releases are "whatever is on main today", there
are no tags, half-finished features ship by accident, and nobody can answer *"what code is in
production right now?"*.

You have been asked to define a **branching strategy**, enforce it with a written workflow, and
prove it by running a **simulated two-week release cycle** end to end: two features, one bug fix,
one hotfix to a released version.

## 2. Objectives

1. Define a lightweight branching model the team will actually follow.
2. Document the commit message and `.gitignore` standards.
3. Execute a full release cycle including an annotated release tag.
4. Execute a **hotfix on a released tag** while feature work continues on `main`.
5. Be able to answer "what is in production" from Git alone.

## 3. Scope

**In scope:** everything in `CT_DevOps_live_Module2` hands-on docs — `git clone/init`, `status`,
`add`, `commit`, `log`, `branch`, `checkout`/`switch`, `merge`, `tag`, `remote`, `push`, `pull`,
`diff`, `stash`, `.gitignore`.

**Out of scope:** CI automation (Module 8), hooks/servers, GitHub Actions.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** SCM-SOP-003 · **Title:** Branching, merging and release tagging

### 4.1 Branch model

| Branch | Purpose | Created from | Merged into | Lifetime |
|---|---|---|---|---|
| `main` | Always-releasable integration branch | — | — | permanent |
| `feature/<ticket>-<slug>` | One ticket of new work | `main` | `main` | days |
| `bugfix/<ticket>-<slug>` | Defect found before release | `main` | `main` | hours–days |
| `release/<x.y>` | Stabilisation for a release | `main` | `main` (back-merge) | weeks |
| `hotfix/<ticket>` | Emergency fix to a shipped version | release tag | `release/<x.y>` **and** `main` | hours |

### 4.2 Rules

1. **No direct commits to `main`.** Everything arrives via a branch merge.
2. **One ticket = one branch.** Branch names carry the ticket ID so Git history maps to Jira.
3. **Commit messages:** `<type>(<scope>): <imperative summary>` where type ∈
   `feat|fix|docs|refactor|test|chore`. Body explains *why*, not *what*.
4. **Never commit secrets or build output.** `.gitignore` is reviewed with every new file type.
5. **Every release is an annotated tag** (`git tag -a`), never a lightweight tag — annotated tags
   carry author, date and message, and that is your audit record.
6. **Semantic versioning:** `vMAJOR.MINOR.PATCH`.
7. Delete merged branches. A long branch list hides the active work.

### 4.3 Release checklist

| # | Step | Evidence |
|---|---|---|
| 1 | All ticket branches merged to `main` | `git log --oneline --graph` |
| 2 | `main` builds/tests clean | build output |
| 3 | Release notes drafted from the log | `git log <prev-tag>..HEAD --oneline` |
| 4 | Annotated tag created and pushed | `git show <tag>` |
| 5 | Tag communicated to the deploy team | ticket comment |

### 4.4 Rollback
Releases roll back by **deploying the previous tag**, never by rewriting history on `main`.
To undo a bad merge that is already pushed, use `git revert -m 1 <merge-sha>` — a new commit that
undoes it — so other clones stay consistent.

---

## 5. Hands-on implementation

### Step 1 — Create the repository

```bash
# On GitHub: create an empty repo "release-workflow-demo" (no README)
mkdir ~/release-workflow-demo && cd ~/release-workflow-demo
git init -b main

git config user.name  "Your Name"
git config user.email "you@example.com"
git config --global core.editor vim
```

### Step 2 — Standards files, then the first commit

```bash
cat > .gitignore <<'EOF'
# build output
target/
dist/
*.jar
*.war
# logs & temp
*.log
tmp/
# secrets - NEVER commit
*.pem
*.key
*.json.credentials
.env
# editor / OS
.idea/
.vscode/
*.swp
.DS_Store
EOF

cat > README.md <<'EOF'
# Release Workflow Demo
Sample app used to demonstrate the team branching and release standard.
EOF

mkdir -p src
echo 'echo "App v0.1.0"' > src/app.sh
chmod +x src/app.sh

git add .
git status
git commit -m "chore(repo): initialise project with gitignore and skeleton app"

git remote add origin https://github.com/<you>/release-workflow-demo.git
git push -u origin main
```

### Step 3 — Feature 1 (DEV-101)

```bash
git switch -c feature/DEV-101-greeting

cat >> src/app.sh <<'EOF'
echo "Hello from the greeting feature"
EOF

git add src/app.sh
git commit -m "feat(app): add greeting output

Customers asked for a friendly banner on start-up.
Refs DEV-101"

git push -u origin feature/DEV-101-greeting
```

### Step 4 — Feature 2 (DEV-102) in parallel

```bash
git switch main
git switch -c feature/DEV-102-version-flag

cat >> src/app.sh <<'EOF'
[ "$1" = "--version" ] && echo "0.2.0"
EOF

git add -A
git commit -m "feat(app): support --version flag

Support team needs a quick way to confirm the deployed build.
Refs DEV-102"
git push -u origin feature/DEV-102-version-flag
```

### Step 5 — Integrate both features

```bash
git switch main

# Fast-forward is fine for the first one...
git merge feature/DEV-101-greeting

# ...force a merge commit for traceability of the second
git merge --no-ff feature/DEV-102-version-flag -m "merge(DEV-102): version flag into main"

git log --oneline --graph --decorate --all
git push origin main

# tidy up
git branch -d feature/DEV-101-greeting feature/DEV-102-version-flag
git push origin --delete feature/DEV-101-greeting feature/DEV-102-version-flag
```

### Step 6 — Cut release v1.0.0

```bash
# Draft release notes straight from history
git log --oneline --no-merges

git tag -a v1.0.0 -m "Release 1.0.0

- DEV-101 greeting banner
- DEV-102 --version flag"

git push origin v1.0.0

git tag -n
git show v1.0.0 | head -20
```

### Step 7 — Feature work continues on main

```bash
git switch -c feature/DEV-103-config
echo 'echo "config loaded"' >> src/app.sh
git add -A && git commit -m "feat(app): load configuration at start-up

Refs DEV-103"
git switch main
git merge --no-ff feature/DEV-103-config -m "merge(DEV-103): config loading into main"
git push origin main
```

### Step 8 — Production incident → hotfix on the released tag

Production runs **v1.0.0**. `main` has already moved on, so you must branch from the **tag**:

```bash
git switch -c hotfix/DEV-150-version-typo v1.0.0
git log --oneline -1            # confirm you are on the released code, not main

sed -i 's/0.2.0/1.0.0/' src/app.sh
git add -A
git commit -m "fix(app): report the correct version string

--version returned 0.2.0 on the 1.0.0 release, breaking the
support team's build verification. Refs DEV-150"

git tag -a v1.0.1 -m "Hotfix 1.0.1 - correct version string (DEV-150)"
git push origin hotfix/DEV-150-version-typo v1.0.1
```

**Then merge the fix forward into `main` so it is not lost in the next release:**

```bash
git switch main
git merge --no-ff hotfix/DEV-150-version-typo -m "merge(DEV-150): hotfix forward-merge into main"
git push origin main
git log --oneline --graph --decorate --all
```

> **Job insight:** forgetting the forward-merge is one of the most common real-world Git
> mistakes — the bug "comes back" in the next release and nobody understands why.

### Step 9 — Answer "what is in production?"

```bash
git tag -n                                  # all releases with messages
git describe --tags                         # nearest tag to HEAD
git log v1.0.0..v1.0.1 --oneline            # what changed between releases
git diff v1.0.0 v1.0.1 -- src/app.sh        # exact code delta
git show v1.0.1                             # who tagged it, when, why
git branch --contains v1.0.1                # which branches carry the fix
```

### Step 10 — Safe undo of a bad merge

```bash
# Simulate: a merge that should not have shipped
git log --oneline --merges -3
# git revert -m 1 <merge-commit-sha>   # creates a NEW commit undoing the merge
# git push origin main
```

---

## 6. Acceptance criteria

- [ ] `.gitignore` blocks build output and credential file patterns
- [ ] At least 3 feature/bugfix branches created, merged and deleted
- [ ] `git log --graph` clearly shows merge commits with ticket references
- [ ] Annotated tags `v1.0.0` and `v1.0.1` exist and are pushed
- [ ] Hotfix was branched **from the tag**, not from `main`, and forward-merged
- [ ] You can produce a release-notes diff between two tags in one command

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `error: failed to push some refs` | Remote has commits you don't | `git pull --rebase origin main` then push |
| Tag pushed but not visible on GitHub | Tags are not pushed by `git push` | `git push origin <tag>` or `--tags` |
| Committed a secret | `.gitignore` added too late | Remove from history, **rotate the credential**, then re-push |
| Merged the wrong branch | — | `git revert -m 1 <merge-sha>` (never rewrite pushed history) |
| Wrong branch for your work | Branched from stale `main` | `git stash`, `git switch -c correct-branch`, `git stash pop` |
| `detached HEAD` after checking out a tag | Expected behaviour | `git switch -c <branch> <tag>` to start work from it |

---

## 8. Deliverables

1. Public repo showing the full branch/merge/tag graph.
2. `BRANCHING-STANDARD.md` — model table, naming, commit convention, release checklist.
3. `RELEASE-NOTES-v1.0.0.md` and `v1.0.1.md` generated from `git log`.
4. Screenshot of `git log --oneline --graph --decorate --all`.

---

## 9. Interview talking points

* *"Describe your branching strategy."* — Give the table, and justify it: short-lived branches,
  `main` always releasable, tags as the production record.
* *"Production is on v1.0.0 but main has new features — how do you ship an urgent fix?"* —
  Branch from the **tag**, fix, tag `v1.0.1`, deploy, then **forward-merge into main**.
* *"Why `--no-ff`?"* — Keeps the feature's commits grouped and makes the merge revertible as a
  single unit.
* *"How do you undo a merge that is already pushed?"* — `git revert -m 1`, never
  `reset --hard` + force push on a shared branch.

**CV bullet:** *Defined and rolled out the team's Git branching, commit-message and semantic
release-tagging standard; implemented a tag-based hotfix and forward-merge process that made
"what is running in production" answerable directly from source control.*
