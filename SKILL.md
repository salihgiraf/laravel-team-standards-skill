---
name: laravel-team-standards
description: Install team coding standards into a Laravel project by dropping config files for multiple coding agents at once — CLAUDE.md for Claude Code, .github/copilot-instructions.md for GitHub Copilot, .cursor/rules/laravel-standards.mdc for Cursor, and AGENTS.md for Codex. Use this skill whenever the user says "set up coding standards", "add team guidelines", "onboard this Laravel project", "configure Claude Code / Copilot / Cursor / Codex for this project", "add coding rules", or any variant that implies preparing a Laravel project so AI coding agents follow the team's architecture (repository + service pattern, auto-binding, split routes, bootstrap/app.php exceptions, naming conventions, anti-patterns). Trigger even if the user only names one agent — they usually want the full set so the project is consistent across tools. Also trigger on `/laravel-standards`.
---

# Laravel Team Standards Installer

This skill drops a set of config files into a Laravel project so every AI coding agent on the team follows the same architecture and conventions — repository + service pattern, auto-binding, split route files, exception handling in `bootstrap/app.php`, naming conventions, and the anti-patterns that block PRs.

## What this skill does

When invoked, this skill writes four files into a Laravel project, one per agent:

| Agent | Destination path |
|---|---|
| Claude Code | `<project-root>/CLAUDE.md` |
| GitHub Copilot | `<project-root>/.github/copilot-instructions.md` |
| Cursor | `<project-root>/.cursor/rules/laravel-standards.mdc` |
| Codex (OpenAI CLI) | `<project-root>/AGENTS.md` |

Each file is self-contained — the agent reads it directly without needing anything else. Content is in `assets/` beside this `SKILL.md`.

## How to run it

1. **Find the project root.** If the user is already inside a Laravel project (there's a `composer.json` with `laravel/framework` nearby, or an `artisan` file), use that directory. Otherwise ask the user for the path.

2. **Check what's already there.** List which of the four destination files already exist. Do not silently overwrite.

3. **Ask the user what they want.** Present a short choice:
   - Which agents to configure (default: all four).
   - For files that exist: overwrite, skip, or append (default: ask file-by-file).

4. **Write the files.** For each chosen agent:
   - Read the matching asset from `assets/` (see table below).
   - Create the destination directory if needed (`.github/`, `.cursor/rules/`).
   - Write the file.
   - On overwrite, keep a backup at `<file>.bak` so the user can diff and recover.

5. **Report.** Summarize in one short line per file: wrote / skipped / overwrote. Then suggest the user commit these in a single commit (`chore: add team coding standards for AI agents`).

## Asset mapping

| Agent | Source asset | Destination (relative to project root) |
|---|---|---|
| Claude Code | `assets/CLAUDE.md` | `CLAUDE.md` |
| GitHub Copilot | `assets/copilot-instructions.md` | `.github/copilot-instructions.md` |
| Cursor | `assets/cursor-rules.mdc` | `.cursor/rules/laravel-standards.mdc` |
| Codex | `assets/AGENTS.md` | `AGENTS.md` |

## Why four files, not one

Each agent has its own convention for where team rules live and how terse they should be. A single shared file (e.g. a symlinked `RULES.md`) doesn't work because:

- **Claude Code** auto-loads `CLAUDE.md` from the project root. Other names are ignored.
- **GitHub Copilot** auto-loads `.github/copilot-instructions.md` and prefers short, imperative rules.
- **Cursor** reads `.cursor/rules/*.mdc` with YAML frontmatter (`description`, `globs`, `alwaysApply`). Plain markdown isn't picked up.
- **Codex CLI** follows the `AGENTS.md` convention at the project root.

So each file speaks the agent's native dialect. The rules are the same; only the format and verbosity differ.

## When the user names one agent, not all

If the user says "add Cursor rules" or "set up Copilot for this project", still *offer* the full set in one line ("I can also drop CLAUDE.md, AGENTS.md, and the Copilot file so your whole team is covered — add those too?"). If they only want the one they asked for, respect that. Don't be pushy.

## When the user wants to update, not install

If the destination files already exist and the user is clearly asking for a refresh ("update our coding standards", "pull in the latest guidelines"), diff the existing file against the asset. Show a short summary of what would change. Overwrite only after they confirm.

## Customization

The files in `assets/` are the canonical versions. If a specific project needs tweaks (e.g. Pest is not used, or the API lives under `/api/v2` instead of `/v1`), edit the installed file in that project — do not edit the asset. The asset is the shared default across all projects.

## Reference

The full, long-form version of these guidelines lives in `references/full-guidelines.md`. The asset files are condensed from that document. Read the reference when you need the full reasoning behind a rule, example code that was trimmed for brevity, or sections that the asset files only mention in passing (testing, security, performance, git workflow).
