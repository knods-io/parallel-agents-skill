---
name: parallel-delivery
description: When the user requests 2+ independent features or fixes in a single message, run them as parallel implementation workers in isolated git worktrees, validate via an integration branch, then promote to staging. The final main merge only happens after the user explicitly authorizes. Each worker gets a mythological name for clarity in status updates.
---

# Parallel Delivery

Coordinated multi-worker pipeline for shipping multiple independent features in one session, with hard isolation per worker and the user as the sole approval gate for production.

## When to trigger

- User asks for **two or more independent features/fixes** in a single message
- The tasks touch **mostly non-overlapping file surfaces** (some shared files are OK — they get reconciled in the integration merge)
- The user expects everything tested and shipped to staging, not just researched

If the user asks for only one feature, do not invoke this skill. If multiple tasks are inter-dependent (Task B needs Task A's output), do them serially instead.

## The roster

Five named workers, each with a fixed role. Always introduce them by name in your initial status message so the user can follow the trail in logs.

| Role | Name | Mission |
|---|---|---|
| Implementer | **Atlas** | Structural work, UI/layout changes, large refactors — carries the weight |
| Implementer | **Argus** | Bug fixes touching multiple files, edge cases, anything requiring cross-file vigilance — the many-eyed |
| Implementer | **Janus** | Migrations, provider swaps, service transitions — god of transitions |
| Validator | **Themis** | Integration merge, type checks, build sanity, orphan/existence greps — the order keeper |
| Smoke tester | **Apollo** | Live tests against the integration branch — the all-seer |

If a batch has fewer or more than three implementation tasks, scale up/down. Reuse Atlas/Argus/Janus first; if a fourth implementer is needed, use **Hermes** (network/API plumbing) or **Hephaestus** (build/tooling). Validator and smoke tester always stay Themis + Apollo. Roles are labels for the log, not jails — a task spanning UI + provider can still belong to one worker.

## Flow

```
                                ┌─→ Atlas   (worktree + branch worker/atlas-*)
[user request]  spawn parallel ─┼─→ Argus   (worktree + branch worker/argus-*)
                                └─→ Janus   (worktree + branch worker/janus-*)
                                       │
                                       ▼
                                   Themis
                              (build integration
                               branch, validate)
                                       │
                                       ▼
                                   Apollo
                              (smoke test integration)
                                       │
                                       ▼
                              fast-forward staging,
                                  push staging
                                       │
                                       ▼
                               user tests in staging
                                       │
                                       ▼
                              user authorizes merge
                                       │
                                       ▼
                              agent ff-merges
                              staging → main, pushes
```

`main` is never touched by the agent flow until the user explicitly authorizes the production merge. All worker commits live on `worker/*` branches in isolated worktrees, get merged into a transient `integration` branch for validation, then fast-forward onto `staging`. Only `staging` is pushed by the flow itself.

## Step-by-step

### 1. Decide and align

Before spawning anything, decide whether you actually need to ask. Ask **only if there is real ambiguity** the workers can't resolve:

- Scope decisions with material consequences (which option / which library / which UX direction)
- Research questions the workers shouldn't have to guess
- Architectural choices with different cost/perf profiles

If you can proceed without asking, **state your assumptions explicitly** in the opening message and start. Don't manufacture ambiguity to feel safe.

When you do need to ask, surface 2-4 named options per question. Don't begin work until answered.

### 2. Spawn implementers in isolated worktrees

Use your agent/sub-agent spawning mechanism with **isolation** (e.g., git worktrees, separate directories) for every implementer. Spawn all workers **concurrently** — they must run in parallel, not serially.

Worktree isolation gives each worker a separate working directory and a separate branch — they literally cannot collide on disk. Tell each worker to commit on a branch named `worker/<name>-<slug>` (e.g. `worker/atlas-assets-panel`).

Each worker prompt must:

- State the mission in one sentence and the worker's name (e.g. "You are Atlas")
- Cite likely files with grep patterns to bootstrap exploration
- Specify constraints:
  - You are in an isolated worktree — commit on your designated branch, don't push, don't touch `main` or `staging`
  - Run your project's type checker / linter at the end and resolve any errors
  - Grep your own changed files for orphans / dead references
  - Return the exact branch name and the worktree path in your final report
- Demand a structured final report (500-700 words max)
- Forbid out-of-scope work explicitly

Then tell the user the branch names so they can follow.

### 3. Wait for all workers to return

Don't poll. Wait for notifications or completion signals from each worker.

Track progress — mark each worker completed as it returns. Capture each worker's reported branch name; you'll need them in step 4.

### 4. Themis — build the integration branch and validate

After all implementers return, run Themis (serial, not background). Themis does:

**a. Sync staging:**
```
git fetch origin
git checkout staging
git pull --ff-only origin staging
```

**b. Create a fresh integration branch from staging:**
```
git branch -D integration 2>/dev/null || true
git checkout -B integration
```

**c. Merge each worker branch with `--no-ff`** so every feature is one merge commit (clean revert surface later):
```
git merge --no-ff worker/atlas-<slug> -m "merge(integration): atlas — <summary>"
git merge --no-ff worker/argus-<slug> -m "merge(integration): argus — <summary>"
git merge --no-ff worker/janus-<slug> -m "merge(integration): janus — <summary>"
```
If a merge fails with a conflict, **stop and report**. Don't auto-resolve.

**d. Validate the merged state:**
- Run your project's type checker / linter
- Orphan grep — anything that should NOT exist anymore (old env vars, removed imports, deleted prop usage)
- Existence grep — anything that SHOULD exist (new env vars, new function definitions, new model IDs)
- Run the build command, check exit code + artifact size
- `git log --oneline staging..integration` to confirm the merge graph

Themis reports status with optional follow-ups. Red means do not promote. Yellow means promote and follow up. Green means clean.

### 5. Apollo — smoke test integration

After Themis greenlights, run Apollo. Apollo tests **against the integration branch** (currently checked out), feature-by-feature:

- **UI features**: navigate via browser automation, screenshot, inspect DOM state
- **API/route features**: hit endpoints to confirm routes exist (400 = wired, 404 = orphan), then a minimal real call
- **Log inspection**: read process output to confirm startup messages and request flow

Apollo reports each track pass/fail with evidence and a final verdict.

### 6. Promote integration to staging

If Apollo is green:

```
git checkout staging
git merge --ff-only integration
git push origin staging
```

Then clean up:

```
git branch -D integration
# remove all worktrees and worker branches
```

`main` is untouched. Worker branches and worktrees are gone. Only `staging` advanced.

### 7. Hand off to the user

Report back:
- Each worker's outcome (one line)
- Each merge commit SHA now on staging
- What the user should test in staging (concrete checklist)
- Pending follow-ups (from Themis/Apollo)
- Confirmation that `main` is untouched, awaiting authorization

Then stop and wait. Don't proceed to step 8 until the user authorizes.

### 8. User-authorized merge to main

When the user authorizes with a clear phrase — "approved", "ship it", "merge to main", or equivalent — execute:

```
git fetch origin
git checkout main
git pull --ff-only origin main
git merge --ff-only staging
git push origin main
```

If `git merge --ff-only staging` fails (staging diverged from main somehow), **stop and report**. Do not `--no-ff` or force.

Report the new `main` SHA and confirm production push.

### 9. Rollback

If the user finds an issue in staging or after a main push, revert at the **feature level** using the no-ff merge commit. Don't chase individual worker commits.

On staging:
```
git checkout staging
git revert -m 1 <merge-sha>
git push origin staging
```

If already on main:
```
git checkout main
git revert -m 1 <merge-sha>
git push origin main
git checkout staging
git merge --ff-only main
git push origin staging
```

`-m 1` keeps the first parent (the host branch's history) and reverts the merged-in feature. Always confirm with the user which merge SHA to revert before executing.

## Anti-patterns to avoid

- **Don't spawn workers without isolation.** Without it, multiple agents writing the same repo at once will collide.
- **Don't spawn workers serially.** Workers go out concurrently in one batch.
- **Don't commit on `main` from the flow.** Workers commit on `worker/*`, Themis merges into `integration`, integration fast-forwards `staging`. `main` only advances after explicit user authorization.
- **Don't push main without the explicit authorization phrase.** This is the production gate; never assume.
- **Don't merge worker branches into staging directly.** The `integration` branch is the validation buffer.
- **Don't auto-resolve merge conflicts during Themis.** Stop and report. Conflicts are a design signal.
- **Don't skip Themis** even if implementers report clean — Themis catches cross-worker conflicts no individual implementer can see.
- **Don't skip Apollo** even if Themis is green — type clean does not mean runtime correct.
- **Don't manufacture ambiguity to feel safe.** Ask only when there's a real, material question. Otherwise state your assumptions and proceed.

## When to update this skill

If a future batch reveals a missing role (e.g. a "DB migration writer" worker), add it to the roster and update the trigger criteria. Keep the file concise so it stays loadable by your LLM.
