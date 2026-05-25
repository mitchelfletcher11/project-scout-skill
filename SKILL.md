---
name: project-scout
description: >
  Use this skill when the user wants to discover the latest resources before
  starting a project, wants to know what's new in the Claude/AI ecosystem,
  or asks about trending repos, new MCP servers, Claude Code plugins, project
  templates, or CLI tools. Accepts an optional use-case hint (e.g.
  /project-scout HTML slides) but always conducts a contextual interview
  first to gather enough detail before searching. Triggers on: "before I
  start", "what's new", "latest resources", "discover tools", "project
  scout", "what MCP servers are available", "trending repos", "Claude
  plugins", "CLAUDE.md template", "I want to build X", "find me tools for".
allowed-tools: WebFetch WebSearch Bash Read Write
arguments: [usecase]
---

# Project Scout

You are helping the user discover the best resources before starting a new project. Work through the phases below in order.

---

## Phase 0 — Anthropic Skills Sync (always run first, silently)

Before anything else, check whether Anthropic has published new skills since the last run.

1. Fetch the current skill list from GitHub:
   - WebFetch `https://api.github.com/repos/anthropics/skills/contents/skills` — parse the JSON array; extract the `name` field from each entry (these are the skill folder names)

2. Read the local snapshot:
   - Bash: `cat ~/.claude/anthropic-skills-snapshot.json 2>/dev/null || echo '{"lastChecked":"never","skills":[]}'`

3. Compare: find any skill names present in the GitHub list but NOT in the snapshot's `skills` array.

4. If new skills are found:
   - For each new skill, download its SKILL.md:
     `Bash: curl -sf "https://raw.githubusercontent.com/anthropics/skills/main/skills/{skill-name}/SKILL.md" -o ~/.claude/skills/{skill-name}/SKILL.md`
   - Update the snapshot file at `~/.claude/anthropic-skills-snapshot.json` with today's date and the full new skill list
   - **Notify the user** before Phase 1 with a banner:

```
🆕 New Anthropic skills detected since last project-scout run ({lastChecked}):
  • {skill-name} — {one-line description from its SKILL.md}
  ...
These have been installed globally to ~/.claude/skills/ and are now available.
```

5. If no new skills: skip the banner entirely and proceed silently to Phase 1.

---

---

## Phase 1 — Qualification Interview

Before running any searches, ask the user all of these questions **in a single message**. Do not ask technical questions (language, framework, open-source preference, etc.) — the user provides domain and goal context only; you derive technical terms from their answers.

If `$ARGUMENTS` was provided, use it to pre-fill question 1 as a starting point and ask the user to confirm or expand.

Ask:

1. **What is the main goal of this project?** What problem does it solve, or what does it create?
2. **Who is it for?** Describe the intended audience — yourself, a specific team, end-users, conference attendees, clients, etc.
3. **What does a successful outcome look like?** For example: a working demo, a polished shareable deliverable, an internal tool, a one-off script, something deployed publicly.
4. **In what context will it be used?** For example: live at an event, in a browser, shared via link, run locally, embedded in another product.
5. **What must it be capable of doing?** Describe required capabilities in plain language — e.g. "must look polished", "needs to be interactive", "must work offline", "needs to display data visually", "needs to pull in live information".

After receiving answers:
- Synthesize a one-sentence description of what the user is building
- Derive the technical search terms (language ecosystem, relevant frameworks, tool types) from their answers — do not ask the user for these
- Confirm: "Got it — searching for **[one-sentence summary]**. Starting resource discovery..."

---

## Phase 2 — Resource Discovery

Run all 5 searches in parallel. Use the derived technical context from Phase 1 to filter and rank results. Replace `[USECASE]` below with the synthesized use-case terms you derived.

### 1. GitHub Repos (most stars first)
- WebFetch `https://github.com/search?q=[USECASE]&sort=stars&order=desc&type=repositories` — most-starred repos matching the use case
- WebFetch `https://github.com/trending?since=weekly` — repos gaining the most stars this week; scan for use-case relevance
- Extract: repo name, total star count, language, one-line description, URL
- Show top 8, sorted by total stars descending; flag anything with 1k+ stars

### 2. Claude Code Plugins & Skills (most relevant + popular)
- WebFetch `https://raw.githubusercontent.com/anthropics/claude-plugins-official/main/registry.json` — official plugin registry; filter for entries relevant to the use case
- WebSearch `site:github.com anthropics claude plugins [USECASE]` — targeted search for community plugins
- Report: name, star count if available, what capability it adds, install command

### 3. MCP Servers & Connectors (highest-rated + relevant)
- WebFetch `https://claude.ai/directory` — Anthropic-curated connectors; filter for use-case relevance
- WebFetch `https://github.com/modelcontextprotocol/servers` — official MCP servers repo; note any matching the use case
- WebSearch `MCP server [USECASE] model context protocol github stars 2025` — community MCP servers
- Report: name, star count, what it integrates, `claude mcp add` command

### 4. Project Templates & CLAUDE.md Patterns (most stars first)
- WebFetch `https://github.com/search?q=[USECASE]+CLAUDE.md&sort=stars&order=desc&type=repositories` — most-starred repos for this use case that include a CLAUDE.md
- WebSearch `[USECASE] starter template boilerplate claude code github most stars 2025` — targeted boilerplate search
- Report: repo name, total stars, stack, URL, notable CLAUDE.md patterns

### 5. CLI Tools (most stars / downloads first)
- WebFetch `https://github.com/search?q=[USECASE]+cli&sort=stars&order=desc&type=repositories` — most-starred CLI tools for this use case
- WebFetch `https://www.npmjs.com/search?q=[USECASE]+cli` — npm CLI packages (include if JS/Node is relevant)
- WebFetch `https://pypi.org/search/?q=[USECASE]+cli` — PyPI CLI packages (include if Python is relevant)
- WebSearch `best cli tools [USECASE] 2025 github awesome` — curated lists and awesome-X repos
- Report: tool name, stars or weekly downloads, what it does, install command (`npm i -g`, `pip install`, `brew install`, `cargo install`, etc.)

---

## Output Format

Present results in this structure:

```
# Project Scout Results
**Building:** [one-sentence summary from interview]
**Context:** [audience + usage context]
**Needs:** [key capabilities derived from interview]

---

## 🔥 Top GitHub Repos
| Repo | ⭐ Stars | Language | Description |
|------|---------|----------|-------------|
...

## 🔌 Claude Code Plugins & Skills
| Name | ⭐ | What it adds | Install |
...

## ⚡ MCP Servers & Connectors
| Name | ⭐ | Integrates with | Add command |
...

## 📋 Project Templates & CLAUDE.md Patterns
| Repo | ⭐ Stars | Stack | Link | Notable patterns |
...

## 🖥️ CLI Tools
| Tool | ⭐ / Downloads | What it does | Install |
...

---

## Recommended Starting Point
[2-3 sentences: the best repo to fork or study, the most useful MCP connector to add, and the most useful CLI — with reasoning tied to what the user said they're building]
```
