<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/readme/hero-dark.svg">
    <img alt="codex-workers: Claude Code orchestrates, Codex workers write, review and test" src="assets/readme/hero-light.svg" width="100%">
  </picture>
</h1>

A Claude Code plugin where Claude Code orchestrates and Codex workers write code, review changes, and run tests. You stay in Claude Code while the coding work runs on your Codex quota.

This is an independent community project, not affiliated with Anthropic or OpenAI.

## How it works

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/readme/flow-dark.svg">
  <img alt="Read project rules, brief Codex writing workers, review and test with a separate worker, then run a final test and report" src="assets/readme/flow-light.svg" width="100%">
</picture>

1. The orchestrator reads project rules first and stops if they forbid delegation or require a particular model. Codex reads `AGENTS.md` but not `CLAUDE.md`, so `CLAUDE.md` rules missing from `AGENTS.md` are copied into each brief, or moved into `AGENTS.md` if you agree.
2. It briefs Codex workers and assigns independent work, using separate worktrees when needed to avoid file conflicts. Each worker is a `codex exec` run in a Claude Code background task.
3. After writing finishes, a separate worker reviews the changes and runs tests with instructions not to change source.
4. The orchestrator runs the project's fastest test once and reports results in the user's language. Follow-ups resume the same worker thread.

Changes stay uncommitted in your working tree, or in sibling `../<repo>-wt-<name>` worktrees on `work/<name>` branches that it creates when parallel writers would touch the same files.

The orchestrator is instructed not to commit, push or deploy unless you ask, and never to use flags that disable the Codex sandbox.

## Requirements

- [Codex CLI](https://github.com/openai/codex) (`npm install -g @openai/codex`) logged in and on `PATH`. No other plugin is needed.
- `bash`, `git`, and `sort -V` for the skill's commands.

## Install

In Claude Code:

```text
/plugin marketplace add opit80/codex-workers
/plugin install codex-workers@codex-workers
```

Without the plugin system: copy `skills/codex-workers` into `~/.claude/skills/` and run `/codex-workers <task>`.

## Usage

```text
/codex-workers:codex-workers <task>
```

Example:

```text
/codex-workers:codex-workers Fix the API's pagination so the final page returns the remaining records, add a regression test, and run the relevant tests. Do not commit.
```

It runs only when invoked explicitly. Workers default to the newest GPT Sol model in your Codex catalog at `high` effort; name another model or effort in the task. Your default Codex login is used (or `CODEX_HOME`, if exported).

### Example run

Excerpts from a real run on a small demo repo, with the task "Fix pagination so the final partial page returns the remaining records, add a regression test, and run the tests. Do not commit."

The writing worker's changes:

```diff
--- a/paginate.js
+++ b/paginate.js
-  const pages = Math.floor(items.length / size);
+  const pages = Math.ceil(items.length / size);
--- a/paginate.test.js
+++ b/paginate.test.js
+assert.deepEqual(paginate([1, 2, 3, 4, 5], 3, 2), [5]);
+assert.deepEqual(paginate([1, 2, 3, 4, 5], 4, 2), []);
```

From the final report:

```text
The task was small, so I used one writing worker and then one separate review/test worker.
review: found nothing wrong. It says the fix is correct and minimal, and the new test lines cover both the partial last page and the empty page after it.
My own final test: npm test passed (ok, exit 0), which matches what both workers reported.
Open items: none. The changes are uncommitted, as the task asked.
```

## Limitations

- Developed and run on Linux (the commands assume `bash`) with Claude Code 2.1.309 and codex-cli 0.164.0; other versions, macOS and Windows are untested. End-to-end runs covered a writing worker, a review worker, two parallel writers in one repository, resuming a worker from another directory, and stopping and resuming a worker; worktrees and the usage-limit path have not been exercised yet.
- The model is picked with `codex debug models`, a Codex debugging command that may change; if it returns nothing, the skill falls back to `gpt-6.1-sol`. The model must be available on your Codex plan; otherwise name another model in the task.
- Workers use your own Codex quota. On a usage limit it stops the affected workers and reports their thread ids and reset time; it never switches accounts, retries in a loop or sends probe requests, and resumes the same threads when you ask after the reset.
- Keep the Claude Code session open while workers run: Claude Code stops background tasks when it exits. In unattended runs (`claude -p`, the Agent SDK, cloud sessions) it also stops them after a time limit, 30 minutes by default.

## Issues

[Report an issue](https://github.com/opit80/codex-workers/issues) and include `claude --version` and `codex --version`.

## License

[MIT](LICENSE), Copyright (c) 2026 opit80.
