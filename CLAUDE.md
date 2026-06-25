# CLAUDE.md

Guidance for AI agents working in this repository.

## What this is

**michi** is a Claude Code plugin for **GitHub-issue-driven development**. Its substance is a
single slash command / Agent Skill, `/issue <number>` (the `michi:issue` skill), which plans a
GitHub issue into small tasks, implements them one commit at a time in a **draft PR**, and watches
that PR through to merge. Runs are resumable — state lives in git and on GitHub.

This is a **prompt/markdown plugin**: there is no application code, no build step, and no test
suite. The "program" is the command prompt itself.

## Layout

- `commands/issue.md` — the command. This is where all behavior lives: a Markdown prompt with
  YAML frontmatter (`description`, `argument-hint`, `allowed-tools`). Editing michi means editing
  this file.
- `.claude-plugin/plugin.json` — plugin manifest (`name`, `version`, `description`, …).
- `.claude-plugin/marketplace.json` — marketplace listing for `/plugin marketplace add`.
- `README.md` — user-facing documentation.
- `LICENSE` — MIT.

## Conventions

- **Keep the three sources of truth in sync.** A behavior change usually touches all of:
  1. `commands/issue.md` (the logic),
  2. `README.md` (the user-facing description),
  3. `version` in `.claude-plugin/plugin.json` (bump it on any behavior change).
- **`allowed-tools` is a hard scope.** The command may only run the `gh`/`git` subcommands and
  tools listed in the `allowed-tools` frontmatter of `commands/issue.md`. Before relying on a new
  command, add it there.
- **No new dependencies / build tooling** unless a real need appears — this repo is intentionally
  just Markdown and JSON.

## Validating a change

There are no tests to run. Validate by review:

- Markdown/JSON is well-formed and the `<!-- michi:plan -->` marker block conventions in
  `commands/issue.md` and `README.md` still agree.
- `plugin.json` and `marketplace.json` stay valid JSON (`gh` and the plugin loader parse them).
- Versions and described behavior are consistent across `commands/issue.md`, `README.md`, and
  `plugin.json`.

## How this repo develops (it dogfoods michi)

Work here is itself issue-driven via michi: branches are named `issue-<n>`, each task is one commit
with `Michi-Task: <id>` and `Michi-Issue: <n>` trailers, and changes land through a draft PR that
`Closes #<n>`. Follow that pattern when contributing.
