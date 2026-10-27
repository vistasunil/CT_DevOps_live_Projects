# Project 04 — Merge Conflict, Rebase & History-Recovery Runbook

| | |
|---|---|
| **Module** | 2 — Git |
| **Level** | L2 — Engineer |
| **Role you play** | DevOps Engineer acting as the team's "Git escalation point" |
| **Scope** | `Git-Merge-Conflicts.md`, `Git-Rebase.md`, `Git-Merge.md`, `Common-Git-Commands.md` |

---

## 1. Business problem

Developers lose hours — and occasionally lose work — on Git: conflicted merges abandoned
half-finished, `git reset --hard` run on the wrong branch, force pushes that erase a colleague's
commits, and rebases that duplicate every commit.

You are asked to become the team's Git escalation point and produce a **recovery runbook**:
how to resolve conflicts deterministically, when to rebase versus merge, and how to get work back
when someone has "deleted everything".

## 2. Objectives

1. Resolve conflicts in three flavours: same-line, add/add, and modify/delete.
2. Perform an interactive rebase to clean a messy branch before review.
3. Recover from four classic disasters using `reflog`, `revert`, `cherry-pick` and `stash`.
4. Publish the rebase-vs-merge decision rule the team must follow.

## 3. Scope

**In scope:** `merge`, conflict markers, `merge --abort`, `rebase`, `rebase -i`,
`rebase --continue/--abort`, `cherry-pick`, `revert`, `reset` (soft/mixed/hard), `reflog`,
`stash`, `diff`, `log`.

**Out of scope:** CI, hooks, submodules, LFS.

---

## 4. Standard Operating Procedure (SOP)

> **SOP ID:** SCM-SOP-004 · **Title:** Conflict resolution and history recovery

### 4.1 The one rule that prevents most incidents

> **Never rewrite history (`rebase`, `commit --amend`, `reset --hard`, `push --force`) on a
> branch that other people have pulled.** Rewriting is for *your own* unpushed/unshared work.
> On shared branches use `git revert`.

If a force push is genuinely required, use `--force-with-lease`, which refuses to overwrite
commits you have not seen.

### 4.2 Rebase vs merge decision rule

| Situation | Use | Why |
|---|---|---|
| Updating **your** feature branch with the latest `main` | `rebase` | Linear history, no noise merges |
| Integrating a finished feature into `main` | `merge --no-ff` | Groups the feature, easy to revert |
| Branch already pushed and a colleague pulled it | `merge` | Rebase would rewrite shared commits |
| Cleaning up messy WIP commits before review | `rebase -i` | Squash/reword/fix-up |
| Taking **one** commit from another branch | `cherry-pick` | Minimal, targeted |

### 4.3 Conflict resolution procedure

| # | Step |
|---|---|
| 1 | **Stop.** Do not run random commands. Run `git status` — it names the files and the operation in progress. |
| 2 | `git diff --name-only --diff-filter=U` — list only the conflicted files. |
| 3 | For each file, open it and read all three parts: `<<<<<<< HEAD` (yours), `=======`, `>>>>>>> other` (theirs). |
| 4 | **Understand the intent of both sides.** If you did not write one side, ask its author — do not guess. |
| 5 | Edit to the correct final content and **delete every conflict marker**. |
| 6 | `git add <file>` to mark resolved. |
| 7 | Build/run the code **before** completing the merge. |
| 8 | `git commit` (merge) or `git rebase --continue` (rebase). |
| 9 | If you are out of your depth: `git merge --abort` / `git rebase --abort`. Nothing is lost. |

### 4.4 Recovery authority

| Scenario | Who may act |
|---|---|
| Local unpushed mistakes | The developer |
| Revert on a shared branch | Developer + reviewer |
| Any force push to `main` | DevOps lead only, with a ticket |
| Restoring deleted branches on the remote | DevOps lead |

---

## 5. Hands-on implementation

### Step 0 — Lab setup

```bash
mkdir ~/git-recovery-lab && cd ~/git-recovery-lab
git init -b main
git config user.name "Your Name"; git config user.email "you@example.com"

cat > app.conf <<'EOF'
app_name=payments
threads=4
timeout=30
log_level=info
EOF
git add . && git commit -m "chore: baseline configuration"
```

### Step 1 — Drill A: same-line conflict (the common one)

```bash
git switch -c feature/perf
sed -i 's/threads=4/threads=16/' app.conf
git commit -am "perf(config): raise worker threads to 16"

git switch main
sed -i 's/threads=4/threads=8/' app.conf
git commit -am "perf(config): raise worker threads to 8"

git merge feature/perf        # CONFLICT
```

**Resolve using the procedure:**

```bash
git status
git diff --name-only --diff-filter=U
cat app.conf                  # inspect the markers
```

```
<<<<<<< HEAD
threads=8
=======
threads=16
>>>>>>> feature/perf
```

```bash
# Decision recorded in the commit message - 12 chosen after load test
sed -i '/^<<<<<<<\|^=======\|^>>>>>>>/d' app.conf
sed -i 's/^threads=8$/threads=12/; /^threads=16$/d' app.conf
cat app.conf

git add app.conf
git commit -m "merge(feature/perf): resolve threads conflict

Both branches tuned threads independently. Chose 12 after
reviewing the load-test result; 16 exhausted memory on e2-micro."
git log --oneline --graph --all
```

### Step 2 — Drill B: add/add conflict

```bash
git switch -c feature/docs
echo "Payments service - team A" > SERVICE.md
git add . && git commit -m "docs: add service description"

git switch main
echo "Payments API - team B" > SERVICE.md
git add . && git commit -m "docs: add service description"

git merge feature/docs        # CONFLICT (add/add)
git status
```

```bash
cat > SERVICE.md <<'EOF'
# Payments Service
Payments API owned jointly by team A and team B.
EOF
git add SERVICE.md
git commit -m "merge(feature/docs): combine both service descriptions"
```

### Step 3 — Drill C: modify/delete conflict

```bash
git switch -c feature/cleanup
git rm app.conf && git commit -m "refactor: drop legacy app.conf"

git switch main
echo "retries=3" >> app.conf
git commit -am "feat(config): add retry setting"

git merge feature/cleanup     # CONFLICT (modify/delete)
git status
```

```bash
# Decide explicitly: keep the file (main still needs it)
git add app.conf              # "keep"
# or: git rm app.conf         # "accept the deletion"
git commit -m "merge(feature/cleanup): keep app.conf - retry setting still required"
```

### Step 4 — Interactive rebase: clean a messy branch

```bash
git switch -c feature/DEV-200-metrics
echo "metrics=on"        >> app.conf && git commit -am "wip"
echo "metrics_port=9090" >> app.conf && git commit -am "wip2"
echo "metrics_path=/m"   >> app.conf && git commit -am "fix typo"
git log --oneline -3
```

```bash
git rebase -i HEAD~3
# pick   <sha> wip
# squash <sha> wip2
# squash <sha> fix typo
# -> then write one proper message:
#    feat(config): expose Prometheus metrics endpoint
#
#    Adds metrics toggle, port and path so the scrape config
#    in Module 10 can target this service. Refs DEV-200.
git log --oneline -1
```

### Step 5 — Rebase a feature branch onto an advanced `main`

```bash
git switch main
echo "region=us-central1" >> app.conf
git commit -am "chore(config): record deployment region"

git switch feature/DEV-200-metrics
git rebase main               # may conflict
# On conflict:
#   fix the file -> git add <file> -> git rebase --continue
#   or bail out  -> git rebase --abort
git log --oneline --graph --all
```

### Step 6 — Disaster recovery drills

**D1 — "I ran `reset --hard` and lost my commits"**

```bash
git log --oneline -3
git reset --hard HEAD~2        # simulate the disaster
git log --oneline -3           # commits gone

git reflog                     # EVERY HEAD move is recorded
git reset --hard HEAD@{1}      # or the exact sha from reflog
git log --oneline -3           # recovered
```

**D2 — "I deleted the wrong branch"**

```bash
git branch -D feature/docs
git reflog | grep -i "feature/docs"
git switch -c feature/docs <sha-from-reflog>
```

**D3 — "I committed to the wrong branch"**

```bash
# commit sits on main but belongs on a feature branch
git log --oneline -1                     # note the sha
git switch -c feature/DEV-201-moved
git cherry-pick <sha>                    # copy it here
git switch main
git reset --hard HEAD~1                  # ONLY safe because it is not pushed yet
```

**D4 — "A bad commit is already pushed to main"**

```bash
git revert <bad-sha>          # new commit that undoes it; history preserved
git push origin main
```

**D5 — "I need to switch branches but my work is half-done"**

```bash
echo "half_done=true" >> app.conf
git stash push -m "DEV-202 WIP retry backoff"
git stash list
git switch main
# ... urgent work ...
git switch feature/DEV-201-moved
git stash pop
```

---

## 6. Acceptance criteria

- [ ] Three conflict types resolved, each with a merge commit explaining the **decision**
- [ ] One `merge --abort` and one `rebase --abort` performed deliberately
- [ ] Three WIP commits squashed into one well-written commit via `rebase -i`
- [ ] Commits recovered from `reflog` after a `reset --hard`
- [ ] A deleted branch restored
- [ ] A pushed commit undone with `revert` (not `reset`)
- [ ] Written rebase-vs-merge rule published to the team

---

## 7. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| Conflict markers reached production | File committed without removing markers | `grep -rn '^<<<<<<<\|^>>>>>>>' .` before committing; add it to code review checklist |
| Rebase conflicts on every commit | Branch far behind `main` | Rebase more often, or merge once instead |
| Duplicate commits after rebase | Rebasing a branch others already merged | Use merge for shared branches |
| `fatal: refusing to merge unrelated histories` | Two independent roots | `git merge --allow-unrelated-histories` (only when you are sure) |
| Colleague's commits vanished | Someone force-pushed | Recover from their `reflog` / another clone; enforce `--force-with-lease` + protected branches |
| "Detached HEAD" after `cherry-pick` of a tag | Checked out a tag | `git switch -c <branch>` |

---

## 8. Deliverables

1. `GIT-RECOVERY-RUNBOOK.md` — conflict procedure, rebase-vs-merge table, five recovery drills.
2. Lab repo containing the conflict merge commits with explanatory messages.
3. Before/after `git log --oneline` screenshots of the interactive rebase.
4. A short "Git safety rules" one-pager for the team (history rewriting, `--force-with-lease`, protected `main`).

---

## 9. Interview talking points

* *"How do you resolve a merge conflict?"* — Give the 9-step procedure and stress
  *understanding both sides* and *testing before committing*.
* *"Rebase or merge?"* — Rebase your own unpublished branch onto `main`; merge `--no-ff` into
  `main`; never rebase shared branches.
* *"Someone deleted commits — what now?"* — `git reflog`, reset/branch to the recovered sha;
  explain that Git rarely loses anything for ~90 days.
* *"Difference between `reset --soft`, `--mixed`, `--hard`?"* — moves HEAD only / also unstages /
  also discards working tree changes (destructive).
* *"How do you prevent this class of problem?"* — Protected branches, review before merge,
  `--force-with-lease`, short-lived branches, frequent integration.

**CV bullet:** *Built and ran the team's Git conflict-resolution and history-recovery runbook,
covering interactive rebase clean-up, reflog-based recovery and safe revert procedures;
eliminated repeat incidents of lost work from force pushes and hard resets.*
