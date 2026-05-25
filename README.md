# /project-scout

A Claude Code skill that discovers the best repos, MCP servers, Claude plugins, templates, and CLI tools before you start a new project.

Type `/project-scout` (or `/project-scout HTML slides`) and it interviews you with 5 questions, then runs 5 parallel searches across GitHub, the MCP registry, the Anthropic skills catalogue, project templates, and CLI package indexes — returning a ranked results table and a recommended starting point tailored to what you're building.

## Install

```bash
mkdir -p ~/.claude/skills/project-scout && \
curl -sf https://raw.githubusercontent.com/mitchelfletcher11/project-scout-skill/main/SKILL.md \
     -o ~/.claude/skills/project-scout/SKILL.md
```

Restart Claude Code, then run `/project-scout` to verify.

## Usage

```
/project-scout
```
Launches the qualification interview from scratch.

```
/project-scout HTML slides
```
Pre-fills the first question with your use-case hint — skips straight to confirming the summary.

## What it does

| Phase | What happens |
|-------|-------------|
| **Sync** | Silently checks for new official Anthropic skills and installs any you don't have |
| **Interview** | Asks 5 questions about your goal, audience, success criteria, context, and required capabilities |
| **Search** | Runs 5 parallel searches — GitHub repos, Claude skills & plugins, MCP servers, project templates, CLI tools |
| **Report** | Returns a ranked table per category with star counts and install commands, plus one recommended starting point |

## Update

Re-run the install command at any time to pull the latest version.
