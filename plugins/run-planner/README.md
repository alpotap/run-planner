# run-planner

A Claude Code plugin that plans a project's next unattended run, then runs it task by task.

## What it does

- **Planning.** When you ask Claude to "perform planning on this project", the `plan-run` skill researches the codebase, writes the findings, asks you for owner decisions (D1, D2, ...), and writes `runs/<NNN-name>/PLAN.md`. Each task is sized S, M or L and assigned to Haiku, Sonnet or Opus. Planning never implements tasks.
- **Running.** `tools/run-planner/run_plan.ps1` runs each open task in its own headless `claude` session, in order. It waits out usage-limit pauses, writes a ledger row per task, and prints the cost per task. When the run ends, it also writes token statistics across all runs, with a trend check that flags abrupt changes.

## What it installs

The first planning session in a project runs `install.ps1`, which copies these into the project:

- `tools/run-planner/`: `run_plan.ps1`, `run_task.ps1`, `new_run.ps1`, `token_stats.ps1`, `VERSION`
- `runs/runner.json` and `runs/PROJECT.md`, created once and never overwritten
- a `.runlogs/` line in `.gitignore`

## What it runs and changes

- Every task runs `claude` in headless mode with `--permission-mode acceptEdits`. File edits are approved automatically, and shell commands (Bash and PowerShell) are pre-authorized. A task can edit files and run commands anywhere in the project, and in any sibling directory listed in `addDirs`.
- Tasks commit their own code in the local git repository, and the runner commits the ledger. `git push` is denied for tasks, so nothing is pushed.
- Logs go to `.runlogs/`, which is git-ignored. Plans and the ledger go to `runs/`.
- The plugin scripts make no network calls themselves. The `claude` sessions call the Claude API, as they normally do.

## Requirements

- Claude Code, signed in
- Windows PowerShell 5.1 or PowerShell 7 (`pwsh`). Windows only: the runner is PowerShell scripts only.
- The target project must be a git repository.

## Use it

1. In your project, ask Claude to perform planning, with your requirements. Review `PLAN.md` before you start a run.
2. Run `.\tools\run-planner\run_plan.ps1`. For multi-day runs, keep the machine awake.

The full guide, with the planning process and customisation options, is in the repository [README](https://github.com/alpotap/run-planner#readme).
