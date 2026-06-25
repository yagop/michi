---
description: GitHub-issue-driven development — plan, implement, push, and track an issue in a draft PR, task by task. Resumable.
argument-hint: <number> [owner/repo]
allowed-tools: >-
  Bash(gh auth status:*),
  Bash(gh repo view:*),
  Bash(gh issue view:*),
  Bash(gh issue comment:*),
  Bash(gh pr create:*),
  Bash(gh pr view:*),
  Bash(gh pr list:*),
  Bash(gh pr ready:*),
  Bash(gh pr checks:*),
  Bash(gh run list:*),
  Bash(gh run view:*),
  Bash(gh api:*),
  Bash(git status:*),
  Bash(git log:*),
  Bash(git branch:*),
  Bash(git checkout:*),
  Bash(git switch:*),
  Bash(git worktree:*),
  Bash(git diff:*),
  Bash(git rev-parse:*),
  Bash(git remote:*),
  Bash(git add:*),
  Bash(git commit:*),
  Bash(git push:*),
  Read, Edit, Write, Grep, Glob, TodoWrite
---

You are running **michi**: GitHub-issue-driven development. Drive the target issue to
completion one small task at a time, tracking the work in a **draft PR**. Runs are
**resumable** — state lives in git and on GitHub, so you can be stopped and re-invoked anytime.

## Sync model (read this first)

Don't keep two checklists in sync — that races and drifts. **One ground truth per fact,
everything else derived:**

- **Plan** (task list + text) → ground truth is the **PR body** marker block, and *only* there.
  The PR is opened up front (via an empty bootstrap commit), so the task list never lives in two
  places. The issue gets a one-line pointer comment; its body is never touched.
- **Progress** (what's done) → ground truth is the **git log**: a task is done iff a commit
  carrying its id exists. A checkbox is only a *cache* of that.
- **TodoWrite** → a throwaway projection, rebuilt each run from `plan ⋈ git log`.

Honor these:

- Each task has a **stable id**, assigned once, never reused or renumbered. Match by id, never
  by line position.
- Commits carry the id in a trailer, so done-ness survives reordering, rewording, and rebases.
- "Sync the plan" means re-rendering **only** the marker block in its current home from derived
  state. Idempotent and best-effort: if a write fails, the next run re-derives from git.

## Arguments

The invocation arguments are: `$ARGUMENTS`

Parse them as whitespace-separated tokens:

- The **first** token is the **issue number** (required). If `$ARGUMENTS` is empty, ask the
  user for the issue number and stop. Below, `<issue>` means this number.
- The **second** token, if present, is `owner/repo`. When given, pass `-R <owner/repo>` to
  **every** `gh` call. Below, `$REPO` means "`-R <owner/repo>` if given, else nothing".

## Live context

- Current branch: !`git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "(not a git repo)"`
- Default branch: !`gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || echo "(unknown)"`
- Has remote: !`git remote 2>/dev/null | head -1 || echo "(none)"`
- Working tree: !`git status --short 2>/dev/null | head -30`
- Worktrees: !`git worktree list 2>/dev/null | head -30`

## Safety — what michi may and won't do

- **May**: create the `issue-<issue>` branch **in a dedicated git worktree** (so your current
  checkout — branch, staged changes, untracked files — is never touched), commit, **push that
  branch** (fast-forward only), open and maintain a **draft PR**, **wait for CI and push bounded
  fix commits** to make it pass, and **mark the PR ready for review** once every task is done and
  CI is green.
- **Never** (needs your explicit action): `--force`/force-push, push the **default branch**,
  **merge** the PR, or **close** the issue. The issue closes automatically via `Closes #<issue>`
  when *you* merge — michi never closes it.
- These guards are partly prompt-enforced (the tool scope can't express "non-force, non-default
  branch"), so treat them as hard rules: refuse to push if the current branch is the default
  branch, and never pass `--force`.
- **One commit per task.** Subject `T<n>: <task summary> (#<issue>)`, plus a `Michi-Task: <id>`
  and `Michi-Issue: <issue>` trailer added natively with `git commit --trailer` (don't hand-format
  the trailer paragraph). The id trailer is how done-ness is derived (§3).
- Make **only the change the current task requires**. Don't batch tasks into one commit.

## 0. Preconditions

1. `gh auth status` — if not authenticated, tell the user to run `gh auth login` and stop.
2. Confirm you're inside a git repo (see Live context). If not, stop and say so.
3. Note whether a remote exists. **No remote → local-only mode** (the exception): a PR isn't
   possible, so keep the plan's marker block in a `😺 Michi` issue comment and tick it there.
   Skip every push and PR step below; everything else is unchanged.

## 1. Read state & locate the plan

1. Fetch the issue (for its title): `gh issue view <issue> $REPO --json number,title,body,state`.
2. Find the work, in order:
   - **PR** (the normal home): `gh pr list $REPO --head issue-<issue> --state all --json number,body,isDraft,state`.
     If one exists → the plan lives in its body. This is a **resume**; go to §3.
   - Else if the **branch** `issue-<issue>` exists with commits but no PR (an interrupted
     bootstrap) → **resume**; go to §3 (it will open the PR).
   - Else (local-only mode) a `😺 Michi` issue comment may hold the plan → **resume** from there.
   - Else → **fresh start**; go to §2.

The marker block looks like:

```
<!-- michi:plan -->
- [ ] T1 <!--m:a1b2--> …
- [x] T2 <!--m:c3d4--> …
<!-- /michi:plan -->
```

Each line's `<!--m:…-->` is its **stable id** — the key everything matches on.

## 2. Plan (fresh start)

1. Read the issue title + body; skim repo conventions (build/test) enough to plan well.
2. Decompose into **small, independently committable tasks** — each a coherent single commit,
   verifiable on its own. Prefer 3–8; split only where it earns its keep.
3. Give each task a **stable id**: a short unique token (4–6 lowercase alphanumeric chars),
   assigned once and never changed.
4. **Set up an isolated worktree** for the issue, so your current checkout is never disturbed —
   no branch switch, no stash. Pick a path *outside* the repo (a sibling of the repo root), then
   create the `issue-<issue>` branch and its worktree off the **default branch** in one step:
   ```
   ROOT="$(git rev-parse --show-toplevel)"; WT="$(dirname "$ROOT")/$(basename "$ROOT")-issue-<issue>"
   git worktree add "$WT" -b issue-<issue> <default-branch>
   cd "$WT"
   ```
   From here on run **every** git command, file read, write, and commit inside `$WT`. (Local-only
   mode still creates the worktree — only push/PR are skipped.)
5. **Open the draft PR up front** (so the plan has a durable home from the start, and the issue
   can point at it). GitHub needs a commit to open a PR, so bootstrap with an empty one:
   ```
   git commit --allow-empty -m "michi: start #<issue>"
   git push -u origin issue-<issue>
   gh pr create $REPO --draft --base <default-branch> --head issue-<issue> \
     --title "<issue title> (#<issue>)" --body-file -
   ```
   The PR body is `Closes #<issue>` then the full `<!-- michi:plan -->` block (all unchecked):
   ```
   Closes #<issue>

   <!-- michi:plan -->
   - [ ] T1 <!--m:<id>--> <task>
   - [ ] T2 <!--m:<id>--> <task>
   <!-- /michi:plan -->
   ```
   (Local-only mode: skip this; put the block in a `😺 Michi` issue comment instead.)
6. **Point the issue at the PR** — post exactly one comment, and do **not** edit the issue body:
   ```
   😺 Michi — started work in <PR url> 🐾
   ```
   `gh issue comment <issue> $REPO --body "😺 Michi — started work in <PR url> 🐾"`.
7. Mirror the tasks into TodoWrite. Proceed to §4.

## 3. Reconcile & resume

The marker block is the plan; **git is the truth for what's done.**

**First, re-enter the worktree.** A prior run created an `issue-<issue>` worktree; find it with
`git worktree list` and `cd` into its path. If the branch exists but its worktree is gone, re-add
one at the same sibling path (§2 step 4) — `git worktree add "$WT" issue-<issue>` (no `-b`; the
branch already exists). Run everything below, and §4, inside it.

Then rebuild from git:

1. Parse the tasks (id + text + box) from the PR body (a `😺 Michi` issue comment in local-only mode).
2. A task is **done iff a commit carries its id** as a trailer. Get all done ids in one call —
   exact, and immune to prose that merely mentions the token (unlike a `--grep` regex):
   ```
   git log <default-branch>..HEAD --format='%(trailers:key=Michi-Task,valueonly)'
   ```
   A task is done iff its id is in that set. The checkbox is not evidence — it's a cache about
   to be refreshed.
3. If a box disagrees with git, **git wins**; note the correction.
4. If the branch has commits but **no PR exists yet** (interrupted bootstrap), open the PR now
   from the recovered plan and post the issue pointer comment (§2 steps 5–6).
5. Rebuild TodoWrite, refresh the PR body so every box matches git (§4 step 5), then resume at
   the **first not-done** task. Proceed to §4.

## 4. Implement (loop, one task at a time)

For each not-done task `T<n>`, in order:

1. Mark it `in_progress` in TodoWrite.
2. **Execute → verify → correct** (loop until green, *then* commit):
   - **Execute**: implement exactly that task. Read the files you need; make the change.
   - **Verify**: run the project's test/build/lint for the touched area if one exists, else
     sanity-check. Let the repo's own hooks run — **never** `--no-verify`.
   - **Correct**: if verify fails, read the output, fix the actual cause, and verify again.
     Repeat — but **bounded** (≈3 corrections). Each fix must be informed by the failure, not a
     blind retry; if attempts stop making progress, bail early.
   - **Don't commit broken code.** If it's still failing after the bound, or the failure is
     **pre-existing / unrelated** to this task, **stop and report** the task as blocked (adjust or
     split the plan, explain why) rather than committing or forcing past it.
3. Once verify is green, commit just this task's changes, adding the trailers natively:
   ```
   git add <the files for this task>
   git commit -m "T<n>: <task summary> (#<issue>)" \
     --trailer "Michi-Task: <id>" --trailer "Michi-Issue: <issue>"
   ```
4. **Push** the issue branch: `git push` (skip in local-only mode; upstream was set in §2).
   **Never** `--force`; **never** the default branch. If the push is **rejected** (non-fast-forward
   → the branch diverged remotely), **stop and report** — do not force past it.
5. **Sync the PR body (idempotent, marker-block only).** The draft PR already exists (§2):
   1. Re-fetch the current body — `gh pr view <PR> $REPO --json body -q .body`.
   2. Replace **only** the text between `<!-- michi:plan -->` and `<!-- /michi:plan -->`, each box
      `[x]` iff a commit carries that task's id (§3 step 2). Leave everything else byte-for-byte.
   3. Write it back with the REST API — **not** `gh pr edit`, which fails on repos with classic
      Projects (it pre-fetches `projectCards` via a deprecated GraphQL field, exits non-zero, and
      doesn't update). Resolve the repo slug (`gh repo view $REPO --json nameWithOwner -q .nameWithOwner`)
      and PATCH:
      ```
      gh api repos/<owner>/<repo>/pulls/<PR> -X PATCH -F body=@<file>
      ```
      (Local-only mode: edit the `😺 Michi` issue comment instead, via `gh issue comment … --edit-last`.)
   4. Mark the task `completed` in TodoWrite. If the write fails, continue — the next run
      re-derives from git and re-syncs.
6. Next not-done task.

If a task turns out wrong-sized or blocked, adjust/split the checklist inside the marker block
(keep ids stable for unchanged tasks), explain why, and continue.

## 5. Wrap up — gate on CI, then ready

When every task is locally green, committed, and pushed, gate the PR on CI before marking it ready.
(Local-only mode has no PR/CI — skip to the final summary.)

1. **Wait for CI** on the PR head: `gh pr checks <PR> $REPO --watch --interval 30` (blocks until all
   checks finish). **No checks configured** → nothing to wait for; go to step 4.
2. **CI green** → go to step 4.
3. **CI red → fix it (bounded loop, ≈3 attempts):**
   1. Find the failures — `gh pr checks <PR> $REPO` (which failed), then read the logs:
      `gh run list $REPO --branch issue-<issue>` to get the run id, `gh run view <id> $REPO --log-failed`.
   2. Diagnose and fix the **actual cause** locally; re-run the relevant verify (§4 step 2) before pushing.
   3. Commit and push the fix — a fix commit, **not** a task (no `Michi-Task` trailer, so done-ness is
      unaffected): `git commit -m "fix: <what> (#<issue>)" --trailer "Michi-Issue: <issue>"` then `git push`.
   4. Wait for CI again (step 1); repeat until green.
   - **Bail — don't loop forever or burn CI:** after ≈3 honest attempts with no progress, or if the
     failure is **infra / flaky / unrelated** to your change (re-run a flaky check *once*, don't chase
     it), **stop**: leave the PR **draft**, comment with the failing checks and what you tried, and hand
     off. Never mark ready with red CI.
4. **Mark the PR ready for review**: `gh pr ready <PR> $REPO`.
5. Post a short summary PR comment: tasks done + commits, and final CI status. Stop. The PR is **ready
   but not merged** — you have not merged it, force-pushed, or closed the issue. Merging it (the PR says
   `Closes #<issue>`) is theirs to do, and that closes the issue.

**Leave the worktree in place** — it keeps runs resumable. Removing it (`git worktree remove <path>`,
then deleting the branch) stays the user's, like merging; michi never tears it down.
