---
name: codex-workers
description: Act as an orchestrator and delegate work to Codex workers only when the user explicitly invokes this skill.
disable-model-invocation: true
argument-hint: <task to perform>
---

# Orchestrator + Codex workers

Any agent reading this skill, regardless of model or session, is the **orchestrator**. Do not do the work yourself:
split it into packages for Codex workers (default: gpt-6.1-sol, reasoning high), collect the results, and return one report.
The task or user may name another model. Use any relevant task context already present in the conversation.

## Task

$ARGUMENTS

## Your role

- Read and follow the working directory's rules first (AGENTS.md / CLAUDE.md). If those rules forbid delegation
  to another model or require a particular model, **do not start any workers**; report this and stop.
- Explore only enough to plan, then split the work into independent packages.
- **Do not write code, review diffs, or run tests yourself**: assign all of them to workers. Once the writing
  workers finish, start one separate worker for review and tests: pass `--write` (builds and tests write files),
  and include "do not change source" in its brief. The only exception: run the project's fastest test command
  yourself once at the end. You may also run `git status --short` to list changed files and confirm nothing was committed; do not read diffs.
- Be efficient: explore only enough to plan, and put your findings in the brief so workers do not repeat the search.
  Give a small task to one worker instead of splitting it.
- If a report is incomplete or contradictory, send follow-up instructions to the same worker instead of starting a new one.
- No external actions (push, deploy, remote databases, PRs) or commits unless the task explicitly requests them.
- Do not end this session or run `/clear` while workers are running: the session-end hook kills its own jobs
  and deletes their records and logs.

## Commands

Prerequisites: Codex CLI must be logged in, and the OpenAI Codex plugin must be installed in Claude Code.

At the start of every Bash call, resolve the plugin path and set the worker environment. Codex uses your default login, or the profile in `CODEX_HOME` if you have exported it.

```bash
P=$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1)
export CODEX_COMPANION_APP_SERVER_ENDPOINT=unix:/nonexistent/no-broker.sock
node -v >/dev/null 2>&1 || export PATH="$(ls -d ~/.nvm/versions/node/*/bin | sort -V | tail -1):$PATH"
```

If `~/.claude/plugins/cache/openai-codex/codex/*/` matches nothing, stop and tell the user to install the OpenAI Codex plugin.

Do not remove the `CODEX_COMPANION_APP_SERVER_ENDPOINT` line: it runs workers without the broker, so each job
starts its own Codex process. This avoids attaching workers to a broker already open in the directory,
another session's shutdown killing a worker, and lingering thread locks. This depends on internal plugin
behaviour and was tested on plugin 1.0.6: if `$P` points to another version, mention it in your report.

The last line falls back to Node from nvm if the system `node` is broken (for example, a missing library after a package update).

Job records are scoped to the `--cwd` directory: if you start a job with `--cwd`, pass the same `--cwd` to
`status`/`result`/`cancel`, otherwise they report "No job found".

The state file has no lock; with parallel jobs, a record may briefly disappear or be incomplete:

- If `status` reports "No job found", ask once more after 3 seconds. Do not restart the job.
- "No job found" from `result` may also mean the job is still running; check `status` first.


1. **Write the brief to a file** (a temporary directory, for example `D=$(mktemp -d)`; use an absolute path).
   Shell state does not persist between separate Bash calls. Record the directory's absolute path in your notes
   and use it in later commands; reassign `D` to that path in every later Bash call that uses it.
   Do not put long text on the command line.
2. **Start the worker in the background**; it immediately returns a job id:
   ```bash
   node "$P/scripts/codex-companion.mjs" task --background --model gpt-6.1-sol --effort high [--write] [--cwd <dir>] --prompt-file "<brief>"
   ```
   The output is a sentence containing the job id (`task-...`); take the id from it. It also mentions `/codex:status`: ignore that and use the `node ... status` and `node ... result` commands below.
   Pass `--write` to a worker that changes files or runs builds/tests; a worker that only reads uses a read-only sandbox.
   Network access is disabled by default for a `--write` worker: install any required dependencies before starting it.
3. **Wait for completion.** In Claude Code, run in the background (`run_in_background`) and wait for the completion
   notification without polling in between. If background execution is unavailable or not allowed, set the Bash timeout to 600000;
   if it does not finish, repeat the same command:
   ```bash
   node "$P/scripts/codex-companion.mjs" status <job-id> [--cwd <dir>] --wait --timeout-ms 540000
   ```
4. **Get the report:** `node "$P/scripts/codex-companion.mjs" result <job-id> [--cwd <dir>]`
5. **Continue the same worker** (its context is preserved): use the thread id (`Codex session ID`) from the result,
   run in the worker's directory, and use `run_in_background`. Replace `<absolute temp dir>` with the recorded path:
   ```bash
   D="<absolute temp dir>"
   codex exec resume <thread_id> --skip-git-repo-check -m gpt-6.1-sol -c model_reasoning_effort="high" -o "$D/last.md" "<follow-up instructions>" < /dev/null > "$D/resume.log" 2>&1
   ```
   If it needs to write, add `-c sandbox_mode="workspace-write"`. When it finishes, read `$D/last.md`; if the exit
   code is not 0, check the last lines of `$D/resume.log`. Do not use `--resume-last`: a partial record could select
   another worker. If you get "already has an active writer", that thread is running in another process; wait
   for it to finish, then retry.
6. **Stop a stuck job:** `node "$P/scripts/codex-companion.mjs" cancel <job-id> [--cwd <dir>]`
7. **Identify a dead job:** if `status --wait` times out twice or returns an empty state, run the following alone,
   as a separate command (other text on the same line can make it match its own shell):
   `pgrep -f "[t]ask-worker .*--job-id <job-id>"`. If there is output, keep waiting. If there is no output, the worker
   is dead: use `cancel`, then `result` to get its thread id, and resume with step 5 if needed.

## When the usage limit is hit

- Detect `usage limit` (for example, "You've hit your usage limit ... try again at 7:04 AM") or `429` in
  `status`, `result`, or `exec` output.
- Stop attempts for the affected workers. Do not retry in a loop and do not send probe requests.
- Report which workers stopped, their thread ids, and the reset time from the message. If no reset time is
  provided, say so instead of guessing.
- After the reset, continue each affected worker through Commands step 5 with its own thread id and the extra
  instruction "Continue from where you left off." For a writing worker, include `-c sandbox_mode="workspace-write"`.

## Parallel workers

- Start independent packages with `--background` without waiting between starts, then wait for each with `status --wait`.
- **Two writing workers must never touch the same file.** The plugin has no protection for this, and other Claude
  sessions cannot see your plan. If file sets overlap, or another session is writing in the same repository,
  give each writing worker its own worktree: `git worktree add ../<repo>-wt-<name> -b work/<name>`, and point `--cwd`
  to that directory. No commits in worktrees either: changes remain uncommitted in the directory and do not
  appear on the branch. Do not merge branches; report each worktree's directory path and branch
  (committing and merging are the user's decision).
- Start only as many workers as there are truly independent packages.

## Brief template

Workers cannot see you or this task; each brief must stand on its own:

1. **Goal:** what to do and why.
2. **Scope:** files/directories to touch and those to leave alone.
3. **Context:** facts you already know that the worker does not need to rediscover (file:line).
4. **Rules:** the working directory's AGENTS.md applies; no external actions or commits.
5. **Done when:** for example, the project's test and build commands pass.
6. **Report format:** changed files, commands run and their results, open questions/risks. Keep it short. State the language the report must be written in.

Add to the review worker's brief: "Inspect changes with `git status --short -- <paths>` and `git diff -- <paths>`;
read `??` files directly. Do not use `git add`."

## Prohibitions

- Do not use `--dangerously-bypass-approvals-and-sandbox` or any other flag that disables the sandbox.
- Do not enable the plugin's review gate (`setup --enable-review-gate`).
- Do not leave the default model/effort unless the task says so.

## Your final report (in the user's language, short)

- What was done; worker, job id, thread id, and outcome for each package.
- Changed files (with directory path and branch name if worktrees were used).
- Review/test worker findings and test/build results as reported by that worker; say so if you did not confirm them with your own final test.
- Your own final test command and its result.
- Open items, risks, and decisions needed from the user.
