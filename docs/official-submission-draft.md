# Official directory submission: draft

Draft for the plugin directory submission form (https://clau.de/plugin-directory-submission). Not submitted. Check every field against the live form before pasting.

## Plugin

- **Name (slug, immutable once published):** `run-planner`
- **Display name:** Run Planner
- **Version:** 1.0.0 (`plugins/run-planner/.claude-plugin/plugin.json`, `plugins/run-planner/skills/plan-run/VERSION`)
- **Author:** alpotap
- **License:** MIT (`LICENSE`)
- **Repository:** https://github.com/alpotap/run-planner
- **Category:** development (confirm against the form's list)

## Short description (one sentence)

Plans a project's next unattended run, with phased tasks sized and assigned to Haiku, Sonnet or Opus, and runs it task by task in fresh headless Claude Code sessions.

## Long description

run-planner has two parts.

1. **Planning.** A skill (`plan-run`) researches the project, writes findings, asks the owner for decisions (D1, D2, ...), and writes `runs/<NNN-name>/PLAN.md`. Each task is sized S/M/L and assigned to a model. The plan ends with a regression gate and an adversary check.
2. **Running.** `tools/run-planner/run_plan.ps1` (installed into the project) runs each open task in its own headless `claude` session, waits out usage-limit pauses, records results in a ledger, and reports token use per task.

Planning never implements tasks. Running is a separate, explicit step.

## What the plugin installs and runs

- Skill: `plugins/run-planner/skills/plan-run/` (SKILL.md, references, templates)
- Scripts copied into the target project's `tools/run-planner/`: `run_plan.ps1`, `run_task.ps1`, `new_run.ps1`, `install.ps1`
- Headless sessions use the lean tool set in `runs/runner.json` by default: Bash, PowerShell, Read, Edit, Write, Grep, Glob. A task can request more tools with `**Tools: default**`.
- Models (defaults, overridable in `runs/runner.json`): `claude-haiku-5-5`, `claude-sonnet-5-5`, `claude-opus-5-5`. Effort: Haiku medium, Sonnet low, Opus default.

## Permissions and risk disclosure (reviewers will look for this)

- The runner runs unattended and can edit files and run shell commands in the target project, and in any directory listed in `addDirs`. It does this with the permissions the user's Claude Code session already has.
- It is designed to keep running for days, waiting out usage limits. Users should review `PLAN.md` before starting a run.
- The runner commits task results to git in the target repository. Confirm this before submitting, because it affects what a reviewer expects: state whether commits are local only and whether anything is pushed.
- It writes nothing outside the project and its `addDirs`. Confirm this is true in the scripts before submitting.
- No network calls are made by the plugin itself beyond what `claude` does.

## Testing

- Install: `claude plugin marketplace add https://github.com/alpotap/run-planner` then `claude plugin install run-planner@run-planner`
- Smoke test: in a git project, ask Claude to "perform planning on this project" with a small requirement, then run `tools/run-planner/run_plan.ps1` on one S task.
- Requirement: Windows PowerShell 5.1 or PowerShell 7 on Windows. Windows only: macOS and Linux are not supported, because the runner is PowerShell scripts only.

## Gaps to close before submitting

- [x] **Scope the plugin to its folder.** Done: plugin is in `plugins/run-planner/` with its own `.claude-plugin/plugin.json`; the marketplace points at it.
- [x] **Add `homepage` and `repository` to `plugin.json`.** Done: homepage, repository and license set.
- [x] **Align descriptions.** `plugin.json` and the marketplace plugin entry now use the same description.
- [x] **Platform support.** Windows only. Stated in the requirements and the README.
- [ ] **Confirm the git behaviour** described under "Permissions and risk disclosure".
- [ ] **Add a plugin README** if the form requires one. The root README covers the same ground but is written for GitHub.
- [ ] **Run the installed plugin** from the marketplace once end to end, not only from the repo.

## Submission notes for the form (optional text)

run-planner is a planning-and-runner pair for unattended Claude Code work. It never runs tasks during planning. Running is a separate command the user starts, and it works through ordinary headless `claude` sessions with the lean tool set. Source is MIT and in one repository.
