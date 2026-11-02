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
  review/test worker with `--write`. Exceptions: run the project's fastest test command once at the end, run
  `git status --short`, and compare `git rev-parse HEAD` with step 0 (if it changed, report it; do not reset).
- If a report is incomplete or contradictory, send follow-up instructions to the same worker (step 5).
- No external actions (push, deploy, remote databases, PRs) or commits unless the task explicitly requests them.
- Never use `--dangerously-bypass-approvals-and-sandbox` or any flag that disables the sandbox, and do not
  enable the plugin's review gate (`setup --enable-review-gate`).
- Do not end the session or run `/clear` while workers run: the session-end hook kills their jobs and deletes
  their records and logs.

## Commands

Shell variables do not persist between Bash calls. Start every Bash call that runs `node` or `codex` with:

```bash
P=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
export CODEX_COMPANION_APP_SERVER_ENDPOINT=unix:/nonexistent/no-broker.sock
node -v >/dev/null 2>&1 || export PATH="$(ls -d ~/.nvm/versions/node/*/bin | sort -V | tail -1):$PATH"
```

If `P` is empty, stop and tell the user to install the OpenAI Codex plugin. Keep the
`CODEX_COMPANION_APP_SERVER_ENDPOINT` line: it gives each job its own Codex process instead of a shared broker
that another session can shut down. It relies on plugin 1.0.6 internals; if `$P` is another version, say so.

Job records are scoped to `--cwd`: pass the same `--cwd` to `status`/`result`/`cancel`. The state file has no
lock, so with parallel jobs a record can briefly vanish: on "No job found", wait 3 seconds and run `status`
once more. Never restart the job. `result` also says "No job found" while a job is still running, so check `status` first.

0. **Before the first worker:**
   - Unless the task or user names a model, use the newest GPT Sol model in this Codex login's
     catalog: run this once and use the slug it prints as `<model>`; if it prints nothing, use `gpt-6.1-sol`
     and say so in the report. Use `high` as `<effort>` unless the task or user names another effort.
     ```bash
     codex debug models 2>/dev/null | grep -oE '"slug": *"gpt-[0-9.]+-sol"' | grep -oE 'gpt-[0-9.]+-sol' | sort -V | tail -1
     ```
   - Run `git rev-parse HEAD` in every directory a worker will write to (for a worktree, right after
     `git worktree add`). Skip a directory that is not a git repository and say so.
1. **Write each brief to a file** in a temp dir (`D=$(mktemp -d)`). Note its absolute path and set `D` to it
   again in every later Bash call that uses it.
2. **Start each worker in the background**; start all independent workers before waiting on any:
   ```bash
   node "$P/scripts/codex-companion.mjs" task --background --model <model> --effort <effort> [--write] [--cwd <dir>] --prompt-file "<brief>"
   ```
   It prints the job id (`task-...`); ignore its `/codex:status` hint and use the commands below. Pass `--write`
   to any worker that changes files or runs builds/tests; others get a read-only sandbox. `--write` workers have
   no network: install dependencies before starting them.
3. **Wait** with `run_in_background`, without polling (if background is unavailable or not allowed, set the Bash timeout to
   600000). If it returns while the job is still running, run it again:
   ```bash
   node "$P/scripts/codex-companion.mjs" status <job-id> [--cwd <dir>] --wait --timeout-ms 540000
   ```
4. **Get the report:** `node "$P/scripts/codex-companion.mjs" result <job-id> [--cwd <dir>]`
5. **Continue the same worker** (context preserved) with the `Codex session ID` from its result as
   `<thread_id>`, using `run_in_background`. Write the instructions to `$D/followup-<thread_id>.md` first.
   `codex exec resume` has no directory option, so `cd` to the worker's `--cwd` (or where you started it):
   ```bash
   cd "<worker dir>" || exit 1
   D="<absolute temp dir>"
   codex exec resume <thread_id> --skip-git-repo-check -m <model> -c model_reasoning_effort="<effort>" -o "$D/resume-<thread_id>.md" - < "$D/followup-<thread_id>.md" > "$D/resume-<thread_id>.log" 2>&1
   ```
   For a writing worker add `-c sandbox_mode="workspace-write"`. Read `$D/resume-<thread_id>.md`; on a non-zero
   exit, check the end of the `.log`. Never use `--resume-last` or `--last`: they can pick another worker's
   thread. "already has an active writer" means the thread is still running elsewhere: wait, then retry.
6. **Stop a stuck job:** `node "$P/scripts/codex-companion.mjs" cancel <job-id> [--cwd <dir>]`
7. **Dead job?** If `status --wait` times out twice or returns an empty state, run this as a command on its
   own: `pgrep -f "[t]ask-worker .*--job-id <job-id>"`. Output: keep waiting. No output: the worker died;
   `cancel`, then `result` for its thread id, and resume with step 5 if needed.

## Usage limit

On `usage limit` (e.g. "You've hit your usage limit ... try again at 7:04 AM") or `429` in any output, stop the
affected workers: no retry loops, no probe requests. Report their thread ids and the reset time from the
message (or that none was given). After the reset, resume each with step 5 and "Continue from where you left off."

## Parallel workers

**Two writing workers must never touch the same file**; the plugin does not prevent it and other Claude sessions
cannot see your plan. If file sets overlap, or another session is writing in the same repository, give each
writing worker its own worktree (`git worktree add ../<repo>-wt-<name> -b work/<name>`) and point `--cwd` to it.
Worktree changes stay uncommitted and off the branch; do not commit or merge.

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

- What was done; worker, job id, thread id, and outcome for each package.
- Changed files (with directory path and branch name if worktrees were used).
- Review/test worker findings and test/build results as reported by that worker; say so if you did not confirm them with your own final test.
- Your own final test command and its result.
- Open items, risks, and decisions needed from the user.
