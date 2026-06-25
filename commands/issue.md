---
description: GitHub-issue-driven development — plan, implement, and sync an issue task by task. Resumable.
argument-hint: <number> [owner/repo]
allowed-tools: >-
  Bash(gh auth status:*),
  Bash(gh issue view:*),
  Bash(gh issue edit:*),
  Bash(gh issue comment:*),
  Bash(gh label list:*),
  Bash(git status:*),
  Bash(git log:*),
  Bash(git branch:*),
  Bash(git checkout:*),
  Bash(git switch:*),
  Bash(git diff:*),
  Bash(git rev-parse:*),
  Bash(git add:*),
  Bash(git commit:*),
  Read, Edit, Write, Grep, Glob, TodoWrite
---

You are running **michi**: GitHub-issue-driven development. Drive the target issue to
completion one small task at a time. Runs are **resumable** — state lives in the issue and
the git history, so you can be stopped and re-invoked at any point.

## Sync model (read this first)

Don't treat the issue checklist and your task list as two stores to keep equal — that races
and drifts. Instead: **one ground truth per fact, everything else derived.**

- **Plan** (the task list + its text) → ground truth is the issue body marker block.
- **Progress** (what's done) → ground truth is the **git log**: a task is done iff a commit
  carrying its id exists. A checkbox is only a *cache* of that.
- **TodoWrite** → a throwaway projection, rebuilt each run from `plan ⋈ git log`.

Consequences you must honor:

- Each task has a **stable id**, assigned once, never reused or renumbered. Match tasks by id,
  never by line position.
- Commits carry the id in a trailer, so done-ness survives reordering, rewording, and rebases.
- Syncing the issue means rewriting **only** the marker block from derived state. It is
  idempotent and best-effort: if a write fails, the next run re-derives from git and self-heals.

## Arguments

The invocation arguments are: `$ARGUMENTS`

Parse them as whitespace-separated tokens:

- The **first** token is the **issue number** (required). If `$ARGUMENTS` is empty, ask the
  user for the issue number and stop. Below, `<issue>` means this number.
- The **second** token, if present, is `owner/repo`. When given, pass `-R <owner/repo>` to
  **every** `gh` call. Below, `$REPO` means "`-R <owner/repo>` if given, else nothing".

## Live context

- Current branch: !`git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "(not a git repo)"`
- Working tree: !`git status --short 2>/dev/null | head -30`

## Hard rules (safety)

- **Never** `git push`, force-push, open a PR, auto-merge, or close the issue. Stop and let
  the user do those explicitly. (The tool scope already forbids them — don't try to work around it.)
- **One commit per task.** Subject `T<n>: <task summary> (#<issue>)`, followed by a trailer
  block (last paragraph, consecutive lines, no blank line between them):
  ```
  Michi-Task: <id>
  Michi-Issue: <issue>
  ```
- Make **only the change the current task requires**. Don't batch tasks into one commit.
- **Write only inside the marker block.** Never touch the human's prose elsewhere in the body.
- **Never store done-ness independently.** A box is checked only to mirror an existing commit.

## 0. Preconditions

1. Confirm `gh` is authenticated: `gh auth status`. If not, tell the user to run `gh auth login` and stop.
2. Confirm you're inside a git repo (see Live context). If not, stop and say so.

## 1. Read state

Fetch the issue:

```
gh issue view <issue> $REPO --json number,title,body,labels,state
```

Look in the body for the michi marker block:

```
<!-- michi:plan -->
- [ ] T1 <!--m:a1b2--> …
- [x] T2 <!--m:c3d4--> …
<!-- /michi:plan -->
```

Each line's `<!--m:…-->` is its **stable id** — the key everything matches on.

- **No marker block** → this is a **fresh start**. Go to §2 (Plan).
- **Marker block present** → this is a **resume**. Go to §3 (Reconcile & resume).

## 2. Plan (fresh start only)

1. Read the issue title + body. If the repo has obvious conventions (build/test commands,
   structure), skim what you need to plan well.
2. Decompose the work into **small, independently committable tasks** — each one a coherent
   change that can be a single commit and verified on its own. Prefer 3–8 tasks; split only
   where it earns its keep.
3. Give each task a **stable id**: a short unique token (4–6 lowercase alphanumeric chars).
   Assign it once; it never changes, even if you later reword or reorder the task.
4. Write the checklist **into the issue body**, inside a fresh marker block, preserving the
   user's original body above it:

   ```
   <existing issue body>

   <!-- michi:plan -->
   - [ ] T1 <!--m:<id>--> <task>
   - [ ] T2 <!--m:<id>--> <task>
   <!-- /michi:plan -->
   ```

   Apply it with `gh issue edit <issue> $REPO --body-file -` (pipe the full new body via stdin, or
   write to a temp file and use `--body-file`). Do **not** lose existing body content.
5. Ensure a working branch exists. If you're on the default branch, create one:
   `git switch -c issue-<issue>` (skip if already on a suitable branch).
6. Best-effort label: if `gh label list $REPO` shows an `in-progress` label, add it with
   `gh issue edit <issue> $REPO --add-label in-progress`. If it doesn't exist, continue silently —
   it's nice-to-have, not required.
7. Mirror the task list into Claude Code's TodoWrite so the user sees live progress.

Then proceed to §4 (Implement).

## 3. Reconcile & resume

The marker block is the plan; **git is the truth for what's done.** Rebuild state from git —
don't trust the boxes:

1. Parse the tasks (id + text + box) from the marker block.
2. A task is **done iff a commit carries its id**:
   `git log --grep "^Michi-Task: <id>"` returns a commit. The checkbox is not evidence — it's a
   cache you're about to refresh.
3. If a box disagrees with git (commit exists but unchecked, or checked with no commit), the
   **git-derived** state wins. Note the correction.
4. Rebuild TodoWrite from this derived state.
5. Refresh the marker block so every box matches git (§4 step 5), then resume at the **first
   not-done** task. Proceed to §4.

## 4. Implement (loop, one task at a time)

For each unchecked task `T<n>`, in order:

1. Mark it `in_progress` in TodoWrite.
2. Implement exactly that task. Read the files you need; make the change.
3. **Verify**: run the project's test/build/lint for the touched area if one exists, or
   otherwise sanity-check the change. Don't move on if it's broken.
4. Commit just this task's changes, with the trailer block (second `-m` is one paragraph, so
   git parses it as trailers):
   ```
   git add <the files for this task>
   git commit -m "T<n>: <task summary> (#<issue>)" -m "Michi-Task: <id>
   Michi-Issue: <issue>"
   ```
5. **Sync (idempotent, marker-block only).** This just re-projects git → the issue:
   1. Re-fetch the current body: `gh issue view <issue> $REPO --json body -q .body`.
   2. Replace **only** the text between `<!-- michi:plan -->` and `<!-- /michi:plan -->` with a
      freshly rendered checklist — each box `[x]` iff a commit carries that task's id (§3 step 2).
      Leave everything outside the markers byte-for-byte untouched.
   3. Write it back with `gh issue edit <issue> $REPO --body-file -`.
   4. Mark the task `completed` in TodoWrite.

   If the write fails, continue anyway — the next run re-derives from git and re-syncs.
6. Move to the next unchecked task.

If a task turns out to be wrong-sized or blocked, update the plan in the issue body
(adjust/split the checklist inside the marker block) and explain why, then continue.

## 5. Wrap up

When every task is checked:

1. Post a short summary comment on the issue: `gh issue comment <issue> $REPO --body "<summary>"` —
   list the tasks done and the commits (`T<n>:` … ) made.
2. Stop. Remind the user the commits are **local**; you have not pushed, opened a PR, or
   closed the issue, and will only do so on their explicit go-ahead.
