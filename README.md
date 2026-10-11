# run-planner

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin-blue.svg)](https://claude.ai/code)

A **Claude Code plugin** that plans a project's next **unattended run** and runs it.

- **Planning.** You say "perform planning on this project" and give the requirements. Claude researches the code, writes the findings, asks you the owner decisions (D1, D2, ...), and writes `runs/<NNN-name>/PLAN.md`. Each task is sized S/M/L and assigned to Haiku, Sonnet, or Opus. The plan ends with a regression gate, an adversary check, and (optionally) documentation.
- **Running.** `tools/run-planner/run_plan.ps1` executes every task in its own fresh headless session, waits out usage limits, writes the ledger, and prints where the tokens went.

---

## Quick Start

### Install from GitHub Marketplace (Recommended)

```bash
# Add this repository as a local marketplace
claude plugin marketplace add https://github.com/alpotap/run-planner

# Install the plugin
claude plugin install run-planner@run-planner
```

> **Why this method?** Same as plugins like [`ouro`](https://github.com/iteramen/ouro). Enables `claude plugin update` for versioned updates.

### Alternative: User Skills Directory

```powershell
# Clone, then copy the skill folder to your user skills
git clone https://github.com/alpotap/run-planner $env:TEMP\run-planner
Copy-Item -Recurse "$env:TEMP\run-planner\plugins\run-planner\skills\plan-run" "$HOME\.claude\skills\run-planner"
```

### Alternative: Single Project Only

```powershell
# In your project root, after cloning as above
Copy-Item -Recurse "$env:TEMP\run-planner\plugins\run-planner\skills\plan-run" .claude\skills\plan-run
```

---

## Requirements

- **Claude Code** (CLI or desktop app), signed in
- **PowerShell**: Windows PowerShell 5.1 or PowerShell 7 (`pwsh`) on Windows. macOS and Linux are not supported: the runner is PowerShell scripts only.
- **Git**: The target project must be a git repository

---

## Usage

### 1. Plan

Open a Claude Code session in your target project (use your strongest model, e.g., Opus):

```
Perform planning on this project. Requirements:
<your requirements, an issue link, or a list of complaints>
```

**First time in a project:** Claude runs the installer automatically:

- Creates `tools/run-planner/` with runner scripts
- Creates `runs/runner.json` and `runs/PROJECT.md` from templates
- Adds `.runlogs/` to `.gitignore`
- Researches your codebase to fill `PROJECT.md` (test commands, build commands, repo structure, known failures)
- Asks you for anything it couldn't prove (recorded as assumptions)

**Planning process:**

1. **Findings** — root causes in code (`file:line` + anchor strings)
2. **Problems nobody mentioned** — tech debt discovered during research
3. **Owner decisions** (D1, D2, ...) — D1 is always *"Update documentation when all phases are complete? Y/n"*
4. **Assumptions** (A1, A2, ...) — things believed but not proven
5. **Phases & tasks** — grouped by dependency, sized S/M/L, assigned to Haiku/Sonnet/Opus
6. **Closing phase** — regression gate, adversary check, optional docs task

Answer decisions in short comments (`D3: b`, `A2 wrong: ...`). Claude revises, commits, and repeats until you confirm. Status line then says **"ready to run"**.

### 2. Run

```powershell
# Run newest run with open tasks to completion (unattended)
.\tools\run-planner\run_plan.ps1

# Resume from a specific task
.\tools\run-planner\run_plan.ps1 -Run 008-billing -From 57.3

# Run a single phase
.\tools\run-planner\run_plan.ps1 -Phase 57

# Watch attended tasks live
.\tools\run-planner\run_plan.ps1 -LiveAttended

# Run one task manually
.\tools\run-planner\run_task.ps1 -Task 57.3 -Run runs/008-billing
```

If a task fails 3 times, the run stops. Re-run the same command to continue from that task.

> **Tip:** For multi-day runs, set "When I close the lid" → "Do nothing" in power settings.

### 3. Review

At the end, the runner prints:

- Blocked tasks
- Tasks marked `ATTENDED` (for your review)
- Adversary report (`runs/<run>/ADVERSARY.md`)
- "Decisions to review" locations
- Cost table: model, minutes, turns, context, cost per task

Full logs in `.runlogs/<run>/` — each task's prompt, event stream, and `usage.csv`.

---

## Update

```bash
# Marketplace install (recommended)
claude plugin marketplace update run-planner
# then restart Claude Code session

# User skills install
# Replace folder in ~/.claude/skills/run-planner
```

Bump `plugins/run-planner/skills/plan-run/VERSION` when scripts change. Next planning session in each project detects the new version and reinstalls `tools/run-planner/`. Never overwrites `runs/runner.json` or `runs/PROJECT.md`.

---

## Uninstall

```bash
# Marketplace install
claude plugin uninstall run-planner@run-planner

# User skills install
# Delete ~/.claude/skills/run-planner
```

Projects keep their `tools/run-planner/` and `runs/`. They still run without the plugin.

---

## Customising

| What                                    | Where                                                                |
| --------------------------------------- | -------------------------------------------------------------------- |
| Model IDs (next generation)             | `runs/runner.json` → `models`, or defaults in `scripts/run_plan.ps1` |
| Effort per model (Sonnet = `low`)       | `runs/runner.json` → `effort`; per-task with `**Effort: medium**`    |
| Sibling repos a task may edit           | `runs/runner.json` → `addDirs` (relative paths, e.g., `"../engine"`) |
| Tools installed but off `PATH`          | `runs/runner.json` → `pathAdd`                                       |
| Project rules, commands, known failures | `runs/PROJECT.md`                                                    |
| Planning process itself                 | `plugins/run-planner/skills/plan-run/SKILL.md` and `references/`     |

---

## Repository Structure

```
run-planner/                       # repository = marketplace
├── .claude-plugin/
│   └── marketplace.json        # Marketplace config; points at the plugin folder below
├── plugins/
│   └── run-planner/            # the plugin
│       ├── .claude-plugin/
│       │   └── plugin.json     # Plugin manifest
│       └── skills/plan-run/
│           ├── SKILL.md        # Planning process (loads on "perform planning on this project")
│           ├── VERSION         # Runner version stamped into projects
│           ├── scripts/        # install.ps1, run_plan.ps1, run_task.ps1, new_run.ps1
│           ├── templates/      # PROJECT.md, runner.json for new projects
│           └── references/     # task-writing, closing-phase, plan-format, runner, project-profile, lessons
├── docs/                       # Notes, including the official directory submission draft
├── README.md
└── LICENSE                     # MIT License
```

---

## License

MIT License — see [LICENSE](LICENSE) for details.