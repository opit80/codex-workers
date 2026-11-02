<h1>
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/readme/hero-dark.svg">
    <img alt="codex-workers: Claude Code orchestrates, Codex workers write, review and test" src="assets/readme/hero-light.svg" width="100%">
  </picture>
</h1>

A Claude Code plugin where Claude Code orchestrates and Codex workers write code, review changes, and run tests.

This is an independent community project, not affiliated with Anthropic or OpenAI.

## How it works

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/readme/flow-dark.svg">
  <img alt="Read project rules, brief Codex writing workers, review and test with a separate worker, then run a final test and report" src="assets/readme/flow-light.svg" width="100%">
</picture>

1. The orchestrator reads project rules first and stops if they forbid delegation or require a particular model. Codex reads `AGENTS.md` but not `CLAUDE.md`, so `CLAUDE.md` rules missing from `AGENTS.md` are copied into each brief, or moved into `AGENTS.md` if you agree.
2. It briefs Codex workers and assigns independent work, using separate worktrees when needed to avoid file conflicts.
3. After writing finishes, a separate worker reviews the changes and runs tests with instructions not to change source.
4. The orchestrator runs the project's fastest test once and reports results in the user's language. Follow-ups resume the same worker thread.

Changes stay uncommitted in your working tree, or in sibling `../<repo>-wt-<name>` worktrees on `work/<name>` branches that it creates when parallel writers would touch the same files.

The orchestrator is instructed not to commit, push or deploy unless you ask, and never to use flags that disable the Codex sandbox.

## Requirements

- Codex CLI logged in and on `PATH`.
- Node.js.
- `bash`, `git`, `pgrep`, and `sort -V` for the skill's commands.
- The OpenAI Codex plugin installed in Claude Code:

  ```text
  /plugin marketplace add openai/codex-plugin-cc
  /plugin install codex@openai-codex
  ```

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

The writing worker's change:

```diff
-  const pages = Math.floor(items.length / size);
+  const pages = Math.ceil(items.length / size);
```

From the final report:

```text
Review/test worker. Verdict: ok.
Changed files: paginate.js, paginate.test.js (uncommitted, in the working tree).
My own final test command: npm test. Result: exit 0, prints ok.
Open items: invalid input (negative page, page 0, size 0) is still unvalidated and untested.
```

## Limitations

- Developed and run on Linux (the commands assume `bash`) with Codex plugin 1.0.6 and codex-cli 0.164.0; other versions, macOS and Windows are untested. End-to-end runs covered a writing worker, a review worker, the final report and resuming a worker from another directory; cancel, parallel workers, worktrees and the usage-limit path have not been exercised yet.
- Relies on Codex plugin internals (the cache path `~/.claude/plugins/cache/openai-codex/codex/`, `scripts/codex-companion.mjs`, and the broker bypass via `CODEX_COMPANION_APP_SERVER_ENDPOINT`); an update may break it, and the orchestrator reports a version mismatch.
- The model is picked with `codex debug models`, a Codex debugging command that may change; if it returns nothing, the skill falls back to `gpt-6.1-sol`. The model must be available on your Codex plan; otherwise name another model in the task.
- Workers use your own Codex quota. On a usage limit it stops the affected workers and reports their thread ids and reset time; it never switches accounts, retries in a loop or sends probe requests, and resumes the same threads when you ask after the reset.
- Keep the Claude Code session open while workers run: ending it or using `/clear` can kill jobs and remove their records.

## Issues

[Report an issue](https://github.com/opit80/codex-workers/issues) and include the Codex plugin version and `codex --version`.

## License

[MIT](LICENSE), Copyright (c) 2026 opit80.
