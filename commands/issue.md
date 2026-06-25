---
description: GitHub-issue-driven development — plan, implement, push, and track an issue in a draft PR, task by task. Resumable.
argument-hint: <number> [owner/repo]
allowed-tools: >-
  Bash(gh auth status:*),
  Bash(gh repo view:*),
  Bash(gh issue view:*),
  Bash(gh issue comment:*),
  Bash(gh issue edit:*),
  Bash(gh label list:*),
  Bash(gh pr create:*),
  Bash(gh pr view:*),
  Bash(gh pr list:*),
  Bash(gh pr edit:*),
  Bash(gh pr ready:*),
  Bash(git status:*),
  Bash(git log:*),
  Bash(git branch:*),
  Bash(git checkout:*),
  Bash(git switch:*),
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

- **Plan** (task list + text) → ground truth is the **PR body** marker block (before the PR
  exists, it lives in a `🐱 Michi` issue comment). The issue *body* is never touched.
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

## Safety — what michi may and won't do

- **May**: create the `issue-<issue>` branch, commit, **push that branch** (fast-forward only),
  open and maintain a **draft PR**, and **mark the PR ready for review** once every task is done.
- **Never** (needs your explicit action): `--force`/force-push, push the **default branch**,
  **merge** the PR, or **close** the issue. The issue closes automatically via `Closes #<issue>`
  when *you* merge — michi never closes it.
- These guards are partly prompt-enforced (the tool scope can't express "non-force, non-default
  branch"), so treat them as hard rules: refuse to push if the current branch is the default
  branch, and never pass `--force`.
- **One commit per task.** Subject `T<n>: <task summary> (#<issue>)`, then a trailer block
  (last paragraph, consecutive lines, no blank line between):
  ```
  Michi-Task: <id>
  Michi-Issue: <issue>
  ```
- Make **only the change the current task requires**. Don't batch tasks into one commit.

## 0. Preconditions

1. `gh auth status` — if not authenticated, tell the user to run `gh auth login` and stop.
2. Confirm you're inside a git repo (see Live context). If not, stop and say so.
3. Note whether a remote exists. **No remote → local-only mode**: skip every push and the PR;
   keep the plan in the `🐱 Michi` issue comment and tick its checkbox there. Everything else
   below is unchanged.

## 1. Read state & locate the plan

1. Fetch the issue: `gh issue view <issue> $REPO --json number,title,body,labels,state`.
2. Find the work, in order:
   - **PR**: `gh pr list $REPO --head issue-<issue> --state all --json number,body,isDraft,state`.
     If one exists → the plan lives in its body. This is a **resume**; go to §3.
   - Else **issue comment**: look for a comment starting `🐱 Michi` containing a
     `<!-- michi:plan -->` block → **resume** from there; go to §3.
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
4. Create the working branch if needed: if you're on the default branch, `git switch -c issue-<issue>`
   (otherwise reuse the current non-default branch).
5. **Announce**: post one issue comment (this is your initial durable plan — do **not** edit the
   issue body):
   ```
   🐱 Michi — started work.

   <!-- michi:plan -->
   - [ ] T1 <!--m:<id>--> <task>
   - [ ] T2 <!--m:<id>--> <task>
   <!-- /michi:plan -->
   ```
   `gh issue comment <issue> $REPO --body-file -`.
6. Mirror the tasks into TodoWrite. Proceed to §4.

The PR is opened in §4 right after the first task is pushed (GitHub needs a commit to open a PR).

## 3. Reconcile & resume

The marker block is the plan; **git is the truth for what's done.** Rebuild from git:

1. Parse the tasks (id + text + box) from the current plan home (PR body, else issue comment).
2. A task is **done iff a commit carries its id**: `git log --grep "^Michi-Task: <id>"` returns
   a commit. The checkbox is not evidence — it's a cache about to be refreshed.
3. If a box disagrees with git, **git wins**; note the correction.
4. If commits exist and are pushed but **no PR exists yet**, open it now (§4 step 6).
5. Rebuild TodoWrite, refresh the plan home so every box matches git (§4 step 7), then resume at
   the **first not-done** task. Proceed to §4.

## 4. Implement (loop, one task at a time)

For each not-done task `T<n>`, in order:

1. Mark it `in_progress` in TodoWrite.
2. Implement exactly that task. Read the files you need; make the change.
3. **Verify**: run the project's test/build/lint for the touched area if one exists, else
   sanity-check. Don't move on if it's broken.
4. Commit just this task's changes, with trailers (second `-m` is one paragraph → git treats it
   as trailers):
   ```
   git add <the files for this task>
   git commit -m "T<n>: <task summary> (#<issue>)" -m "Michi-Task: <id>
   Michi-Issue: <issue>"
   ```
5. **Push** the issue branch (skip in local-only mode). First push sets upstream:
   `git push -u origin issue-<issue>` (thereafter `git push`). **Never** `--force`; **never** the
   default branch. If the push is **rejected** (non-fast-forward → the branch diverged remotely),
   **stop and report** — do not force past it.
6. **Ensure the draft PR exists.** If there's no PR yet and the branch now has a pushed commit:
   ```
   gh pr create $REPO --draft --base <default-branch> --head issue-<issue> \
     --title "<issue title> (#<issue>)" --body-file -
   ```
   The body is `Closes #<issue>` followed by the `<!-- michi:plan -->` block. Once the PR exists,
   it is the plan's home; stop updating the issue comment.
7. **Sync (idempotent, marker-block only).** Re-render the plan and write it back:
   1. Re-fetch the current body — `gh pr view <PR> $REPO --json body -q .body` (or the issue
      comment in local-only mode).
   2. Replace **only** the text between `<!-- michi:plan -->` and `<!-- /michi:plan -->`, each box
      `[x]` iff a commit carries that task's id (§3 step 2). Leave everything else byte-for-byte.
   3. Write it back: `gh pr edit <PR> $REPO --body-file -` (or `gh issue comment … --edit-last`
      in local-only mode).
   4. Mark the task `completed` in TodoWrite. If the write fails, continue — the next run
      re-derives from git and re-syncs.
8. Next not-done task.

If a task turns out wrong-sized or blocked, adjust/split the checklist inside the marker block
(keep ids stable for unchanged tasks), explain why, and continue.

## 5. Wrap up

When every task is done:

1. **Mark the PR ready for review**: `gh pr ready <PR> $REPO`.
2. Post a short summary as a PR comment: the tasks done and their commits (`T<n>:` …).
3. Stop. Tell the user the PR is **ready but not merged** — you have not merged it, force-pushed,
   or closed the issue. Merging it (the PR says `Closes #<issue>`) is theirs to do, and that is
   what closes the issue.
