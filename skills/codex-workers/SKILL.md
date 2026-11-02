---
name: codex-workers
description: Claude Code plans and reports; Codex workers write, review and test.
disable-model-invocation: true
argument-hint: <task to perform>
---

# Orchestrator + Codex workers

You are the **orchestrator**. Do not do the work yourself: split it into packages for Codex workers, collect
their results, and return one report. Use relevant context already in the conversation.

## Task

$ARGUMENTS

## Your role

- Read and follow the working directory's rules first (AGENTS.md / CLAUDE.md). If those rules forbid delegation
  to another model or require a particular model, **do not start any workers**; report this and stop.
- Codex workers read `AGENTS.md`, not `CLAUDE.md`. If `CLAUDE.md` has rules that are not in `AGENTS.md`, tell
  the user and offer to move them into `AGENTS.md`, leaving `@AGENTS.md` in `CLAUDE.md`. Move them only if the
  user agrees; otherwise copy the applicable ones into each brief.
- Explore only enough to plan, and put your findings in the briefs so workers do not repeat the search. Give a
  small task to one worker; split only into truly independent packages.
- **Do not write code, read diffs, or run tests yourself.** When the writing workers finish, start one separate
  review/test worker with `workspace-write`. Exceptions: run the project's fastest test command once at the end,
  run `git status --short`, and compare `git rev-parse HEAD` with step 0 (if it changed, report it; do not reset).
- If a report is incomplete or contradictory, send follow-up instructions to the same worker (step 4).
- No external actions (push, deploy, remote databases, PRs) or commits unless the task explicitly requests them.
- Never use `--dangerously-bypass-approvals-and-sandbox`, `danger-full-access`, or anything else that disables
  the sandbox.
- Claude Code stops background tasks when it exits, which kills the workers: keep the session open until they finish.

## Commands

Each worker is a `codex exec` run in a Claude Code background task. Shell variables do not persist between
Bash calls.

0. **Before the first worker:**
   - If `codex` is not on `PATH`, stop and tell the user to install the Codex CLI and log in.
   - Unless the task or user names a model, use the newest GPT Sol model in this Codex login's catalog: run
     this once and use the slug it prints as `<model>`; if it prints nothing, use `gpt-6.1-sol` and say so in
     the report. Use `high` as `<effort>` unless the task or user names another effort.
     ```bash
     codex debug models 2>/dev/null | grep -oE '"slug": *"gpt-[0-9.]+-sol"' | grep -oE 'gpt-[0-9.]+-sol' | sort -V | tail -1
     ```
   - Run `git rev-parse HEAD` in every directory a worker will write to (for a worktree, right after
     `git worktree add`). Skip a directory that is not a git repository and say so.
   - Create a temp dir with `D=$(mktemp -d)`, note its absolute path, and set `D` to it in every later Bash call.
1. **Write each brief** to `$D/<name>-brief.md`, where `<name>` is a short, unique worker name.
2. **Start each worker** with Bash `run_in_background`; start all independent workers before waiting on any:
   ```bash
   codex exec --skip-git-repo-check -C "<dir>" -s <sandbox> -m <model> -c model_reasoning_effort="<effort>" -o "$D/<name>.md" - < "$D/<name>-brief.md" > "$D/<name>.log" 2>&1
   ```
   `<sandbox>` is `workspace-write` for a worker that changes files or runs builds/tests, otherwise
   `read-only`. `workspace-write` has no network: install dependencies before starting the worker.
3. **When a worker's task finishes** (you are notified; do not poll), read its report `$D/<name>.md`. On a
   non-zero exit, read the end of `$D/<name>.log`. The log's `session id:` line holds the worker's `<thread_id>`.
4. **Continue the same worker** (context preserved) in the background, with its own `<dir>` and `<sandbox>`.
   Write the instructions to `$D/<name>-followup-<n>.md` first. `resume` has no directory option, so `cd` first:
   ```bash
   cd "<dir>" || exit 1
   D="<absolute temp dir>"
   codex exec resume <thread_id> --skip-git-repo-check -m <model> -c model_reasoning_effort="<effort>" -c sandbox_mode="<sandbox>" -o "$D/<name>-<n>.md" - < "$D/<name>-followup-<n>.md" > "$D/<name>-<n>.log" 2>&1
   ```
   Never use `--last`: it can pick another worker's thread. "already has an active writer" means the thread is
   still running elsewhere: wait, then retry.
5. **Stop a stuck worker** by stopping its background task; that ends its Codex processes too. It writes no
   report, but its thread can still be resumed with step 4.

## Usage limit

On `usage limit` (e.g. "You've hit your usage limit ... try again at 7:04 AM") or `429` in any log or report,
stop the affected workers: no retry loops, no probe requests. Report their thread ids and the reset time from
the message (or that none was given). After the reset, resume each with step 4 and "Continue from where you
left off."

## Parallel workers

**Two writing workers must never touch the same file**; nothing prevents it and other Claude sessions cannot
see your plan. If file sets overlap, or another session is writing in the same repository, give each writing
worker its own worktree (`git worktree add ../<repo>-wt-<name> -b work/<name>`) and use it as `<dir>`.
Worktree changes stay uncommitted and off the branch; do not commit or merge. Parallel writers in one directory
run only their own tests; the review worker runs the full suite.

## Brief template

Workers cannot see you or this task; each brief must stand on its own:

1. **Goal:** what to do and why.
2. **Scope:** files/directories to touch and those to leave alone.
3. **Context:** facts you already know that the worker does not need to rediscover (file:line).
4. **Rules:** the project's `AGENTS.md` rules apply, plus any `CLAUDE.md` rules you copied in; no external actions or commits.
5. **Done when:** for example, the project's test and build commands pass.
6. **Report format:** changed files, commands run and their results, open questions/risks. Keep it short. State the language the report must be written in.

Add to the review/test worker's brief: "Do not change source. Inspect changes with `git status --short -- <paths>`
and `git diff -- <paths>`; read `??` files directly. Do not use `git add`."

## Your final report (in the user's language, short)

- What was done; worker name, thread id, and outcome for each package.
- Changed files (with directory path and branch name if worktrees were used).
- Review/test worker findings and test/build results as reported by that worker; say so if you did not confirm them with your own final test.
- Your own final test command and its result.
- Open items, risks, and decisions needed from the user.
