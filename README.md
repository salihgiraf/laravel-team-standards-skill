# Laravel Team Standards

A skill that installs our team's Laravel coding standards (repository + service pattern, auto-binding, split routes, exception handling in `bootstrap/app.php`, naming conventions, anti-patterns) so every AI coding agent follows the same rules.

Each agent reads its rules from a different file:

| Agent | File in your Laravel project | Source in this repo |
|---|---|---|
| Claude Code | `CLAUDE.md` | [assets/CLAUDE.md](assets/CLAUDE.md) |
| GitHub Copilot | `.github/copilot-instructions.md` | [assets/copilot-instructions.md](assets/copilot-instructions.md) |
| Codex | `AGENTS.md` | [assets/AGENTS.md](assets/AGENTS.md) |
| Cursor | `.cursor/rules/laravel-standards.mdc` | [assets/cursor-rules.mdc](assets/cursor-rules.mdc) |

The long-form guidelines are in [references/full-guidelines.md](references/full-guidelines.md).

## Install with npx (recommended)

Uses the [Skills CLI](https://skills.sh/). Needs Node.js. Run it from anywhere:

```bash
npx skills add salihgiraf/laravel-team-standards-skill
```

The CLI asks which agents to install the skill for (Claude Code, Codex, GitHub Copilot, Cursor, and others it detects) and whether to install it for this project or globally.

Skip the prompts:

```bash
# Global (all projects), every detected agent
npx skills add salihgiraf/laravel-team-standards-skill -g -y

# Only specific agents
npx skills add salihgiraf/laravel-team-standards-skill -a claude-code -a codex -a github-copilot -a cursor
```

Then open your Laravel project in the agent and say:

> set up coding standards

or run `/laravel-standards`. The skill writes the config file for each agent into the project (`CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`, `.cursor/rules/laravel-standards.mdc`).

Keep it up to date:

```bash
npx skills check
npx skills update
```

Run `npx skills add --help` to see the flags your CLI version supports.

If you'd rather not use npx, use the manual steps below.

---

## Manual install

First, get a copy of this repo:

```bash
git clone https://github.com/salihgiraf/laravel-team-standards-skill.git
```

Below, `SRC` means the folder you cloned into, and you run the commands from the root of your Laravel project.

### Claude Code

#### Install as a skill

Put the repo in Claude Code's skills folder.

**For all your projects:**

```bash
git clone https://github.com/salihgiraf/laravel-team-standards-skill.git ~/.claude/skills/laravel-team-standards
```

PowerShell:

```powershell
git clone https://github.com/salihgiraf/laravel-team-standards-skill.git "$HOME\.claude\skills\laravel-team-standards"
```

**For one project only:**

```bash
git clone https://github.com/salihgiraf/laravel-team-standards-skill.git .claude/skills/laravel-team-standards
```

Restart Claude Code, open your Laravel project, and say:

> set up coding standards

or run `/laravel-standards`. The skill finds the project root, shows which files already exist, asks what to configure, and writes them (keeping a `.bak` backup if it overwrites anything).

#### Install without the skill

Copy the file only:

```bash
cp SRC/assets/CLAUDE.md CLAUDE.md
```

Claude Code auto-loads `CLAUDE.md` from the project root.

---

### GitHub Copilot

Copilot reads `.github/copilot-instructions.md`, so installing means copying that file.

```bash
mkdir -p .github
cp SRC/assets/copilot-instructions.md .github/copilot-instructions.md
```

PowerShell:

```powershell
New-Item -ItemType Directory -Force .github | Out-Null
Copy-Item SRC\assets\copilot-instructions.md .github\copilot-instructions.md
```

Commit the file so the whole team gets it. Copilot Chat in VS Code picks it up automatically (make sure the setting **Use Instruction Files** is on).

---

### Codex

Codex reads `AGENTS.md` from the project root.

```bash
cp SRC/assets/AGENTS.md AGENTS.md
```

PowerShell:

```powershell
Copy-Item SRC\assets\AGENTS.md AGENTS.md
```

Run `codex` from the project root and it will load the file.

---

### Cursor

Cursor reads `.cursor/rules/*.mdc` (the `.mdc` frontmatter is required).

```bash
mkdir -p .cursor/rules
cp SRC/assets/cursor-rules.mdc .cursor/rules/laravel-standards.mdc
```

PowerShell:

```powershell
New-Item -ItemType Directory -Force .cursor\rules | Out-Null
Copy-Item SRC\assets\cursor-rules.mdc .cursor\rules\laravel-standards.mdc
```

---

### Install for all agents at once

Easiest: use the Claude Code skill above and choose all four agents when asked.

Manual, bash:

```bash
cp SRC/assets/CLAUDE.md                CLAUDE.md
cp SRC/assets/AGENTS.md                AGENTS.md
mkdir -p .github .cursor/rules
cp SRC/assets/copilot-instructions.md  .github/copilot-instructions.md
cp SRC/assets/cursor-rules.mdc         .cursor/rules/laravel-standards.mdc
```

Manual, PowerShell:

```powershell
Copy-Item SRC\assets\CLAUDE.md CLAUDE.md
Copy-Item SRC\assets\AGENTS.md AGENTS.md
New-Item -ItemType Directory -Force .github, .cursor\rules | Out-Null
Copy-Item SRC\assets\copilot-instructions.md .github\copilot-instructions.md
Copy-Item SRC\assets\cursor-rules.mdc .cursor\rules\laravel-standards.mdc
```

Then commit them together:

```bash
git add CLAUDE.md AGENTS.md .github/copilot-instructions.md .cursor/rules/laravel-standards.mdc
git commit -m "chore: add team coding standards for AI agents"
```

---

## Verify

Open the project in each agent and ask: *"What architecture pattern does this project use?"* Each should answer with the repository + service pattern.

## Customizing

Edit the installed file inside the project (for example, if a project doesn't use Pest or its API lives under `/api/v2`). Don't edit the files in `assets/`, since those are the shared defaults for every project.

## Updating

Update the skill, then run it again and choose overwrite (or say "update our coding standards"). It shows a diff before replacing anything.

```bash
# Installed with npx
npx skills update

# Installed by cloning
git -C ~/.claude/skills/laravel-team-standards pull
```

If you installed by copying files, copy them again from the updated repo.
