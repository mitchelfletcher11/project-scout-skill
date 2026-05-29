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

1. Fetch the current skill list from GitHub (authenticated):
   - Bash: `curl -sf -H "Authorization: Bearer $(cat ~/.claude/github-token 2>/dev/null)" "https://api.github.com/repos/anthropics/skills/contents/skills"` — parse the JSON array; extract the `name` field from each entry

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
- Note the **runtime architecture** explicitly (e.g. "local Python server", "browser-only", "cloud-deployed Node app") — this is used in Phase 4 to filter applicability
- Confirm: "Got it — searching for **[one-sentence summary]**. Starting resource discovery..."

---

## Phase 2 — Credential Setup (silent, runs before all searches)

Load stored API keys via Bash before any search runs:

```bash
GITHUB_TOKEN=$(cat ~/.claude/github-token 2>/dev/null)
LIBRARIES_KEY=$(cat ~/.claude/libraries-io-key 2>/dev/null)
SO_KEY=$(cat ~/.claude/stackoverflow-key 2>/dev/null)
```

If a key file is missing, fall back gracefully for that source — do not surface the error to the user unless all keys are absent.

---

## Phase 3 — Resource Discovery

Run all searches in parallel where sources are independent. Replace:
- `[USECASE]` — primary URL-encoded search term derived from Phase 1
- `[USECASE2]` — a variant search term for broader coverage
- `[TAGS]` — comma-separated Stack Overflow tags for this stack (e.g. `python,ffmpeg,flask`)
- `[KEY-PACKAGES]` — the 3–5 most critical packages for this stack

---

### 1. GitHub Repos (authenticated REST API — exact star counts, no rate limit)

Use Bash+curl to pass the Authorization header (WebFetch cannot set headers):

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]&sort=stars&order=desc&per_page=10"
```

Run two queries with different search terms for broader coverage. Parse the JSON `items` array:
- `full_name` → repo name
- `stargazers_count` → exact star count
- `language` → primary language
- `description` → description
- `html_url` → URL
- `pushed_at` → last commit date — flag repos not updated in >12 months as potentially stale

Show top 8 results sorted by stars descending. Flag anything with 1k+ stars prominently.

**After identifying the top 2 repos by stars, read their READMEs:**

```
WebFetch https://raw.githubusercontent.com/{owner}/{repo}/main/README.md
```

If `main` returns a 404, try `master`. From each README extract and record internally (for use in Phase 4 synthesis):
- Architecture decisions made (e.g. how they structured routes, what they used for async, how they called the API)
- Key dependencies actually used (may differ from what you'd assume)
- Any documented gotchas, known issues, or limitations sections
- Any performance notes or scaling warnings

Do not dump the full README into the output — use these notes only in Phase 4.

---

### 2. Package Health (PyPI + npm + Libraries.io)

**For Python stacks** — fetch each key package:
```
WebFetch https://pypi.org/pypi/{package}/json
```
Extract: `info.version`, `info.summary`, `urls[0].upload_time` (last release date). Flag packages with no release in >12 months.

**For JS/Node stacks:**
```
WebFetch https://registry.npmjs.org/-/v1/search?text=[USECASE]&size=8
```
Extract: `objects[].package.name`, `objects[].package.description`, `objects[].downloads.weekly`.

**Libraries.io SourceRank** — health score across all ecosystems (0–30 scale):
```bash
curl -s "https://libraries.io/api/search?q=[USECASE]&api_key=$LIBRARIES_KEY&per_page=8&sort=rank"
```
Extract per result: `name`, `rank` (SourceRank), `stars`, `latest_release_published_at`, `normalized_licenses`, `platform`.
- Rank ≥20: healthy and active
- Rank 10–19: acceptable
- Rank <10 or no release in >18 months: flag as risky

---

### 3. Community Insights (HN Algolia + Stack Overflow)

**Hacker News — Show HN posts** (surfaces cutting-edge tools often before they're widely starred):
```
WebFetch https://hn.algolia.com/api/v1/search?query=[USECASE]&tags=show_hn&hitsPerPage=8
```
Extract: `hits[].title`, `hits[].points`, `hits[].num_comments`, `hits[].created_at`, `hits[].url`. Show top 3 by points.

**Stack Overflow — top voted questions** (reveals real pain points in your exact stack before you hit them):
```bash
curl -s "https://api.stackexchange.com/2.3/search?order=desc&sort=votes&intitle=[USECASE]&tagged=[TAGS]&site=stackoverflow&key=$SO_KEY&pagesize=5"
```
Extract: `items[].title`, `items[].score`, `items[].answer_count`, `items[].link`. Show top 3 as "Watch out for" warnings — these are the problems other developers hit with this exact stack.

---

### 4. Claude Code Plugins & Skills

- WebFetch `https://raw.githubusercontent.com/anthropics/claude-plugins-official/main/registry.json` — official plugin registry; filter for entries relevant to the use case
- WebSearch `site:github.com anthropics claude plugins [USECASE]` — community plugins
- Report: name, star count if available, what capability it adds, install command

---

### 5. MCP Servers & Connectors (authenticated GitHub API for exact star counts)

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+mcp+server&sort=stars&order=desc&per_page=8"
```
- WebFetch `https://github.com/modelcontextprotocol/servers` — official MCP servers repo; note any matching the use case
- Collect all results — applicability filtering happens in Phase 4

---

### 6. Project Templates & CLAUDE.md Patterns

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+CLAUDE.md+in:path&sort=stars&order=desc&per_page=8"
```
- WebSearch `[USECASE] starter template boilerplate claude code github 2025`
- Report: repo name, exact stars, stack, URL, notable CLAUDE.md patterns

---

### 7. CLI Tools

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/search/repositories?q=[USECASE]+cli&sort=stars&order=desc&per_page=8"
```
- WebFetch `https://registry.npmjs.org/-/v1/search?text=[USECASE]+cli&size=5` (if JS relevant)
- WebFetch `https://pypi.org/search/?q=[USECASE]+cli` (if Python relevant)
- WebSearch `best cli tools [USECASE] 2025 github awesome`
- Report: tool name, exact stars or weekly downloads, what it does, install command

---

## Phase 4 — Applicability Filter + Synthesis (runs after all Phase 3 searches complete)

This phase reasons over all gathered results before producing output. Do not skip it.

### Applicability Filter

Using the runtime architecture noted in Phase 1, evaluate every MCP server, plugin, and CLI tool found:

**MCP servers** — ask for each: "Can this server be called at runtime from inside a [Flask app / Node server / browser / etc.]?" MCP servers are invoked by Claude Code during development sessions — they are NOT importable libraries. A Flask app cannot call an MCP server at runtime. Tag each result:
- `dev-time` — useful during Claude Code development sessions only (most MCP servers fall here)
- `runtime` — exposes an HTTP or SDK interface the app itself can call

Only show `runtime`-tagged MCP servers in the MCP section of output. Move `dev-time` ones to a separate "Useful during development" note, or omit if none are relevant.

**CLI tools** — tag each:
- `build-time` — runs during setup/installation only
- `dev-time` — useful during development but not shipped
- `runtime` — called by the app itself (e.g. `ffmpeg` as a subprocess)

**Plugins** — same logic: is this callable from within the app, or only from a Claude Code session?

### README Pattern Extraction

Using the README notes gathered in Phase 3 Section 1, identify 2–3 patterns from real implementations that are directly portable to this project. Note gotchas that are not obvious from the library documentation.

### Synthesis

Before writing output, internally draft the "Design Decisions" section (see output format below). This is the most important part of the output — it must:
- Reference specific findings from the search (package names, SO question titles, repo names, README gotchas)
- Name concrete decisions the user should make differently because of what was found
- Be tied directly to the stated architecture and goals from Phase 1
- Not be generic advice that applies to any project

A good design decision entry looks like:
> **Use `ffmpeg-python` not raw subprocess calls** — SourceRank 22, active releases, and sport-video-AI-analysis uses it exactly this way. The README shows their route structure for async render jobs which maps directly to your pipeline.

A bad design decision entry looks like:
> **Choose a well-maintained package** — make sure your dependencies are up to date.

---

## Output Format

```
# Project Scout Results
**Building:** [one-sentence summary from interview]
**Context:** [audience + usage context]
**Needs:** [key capabilities derived from interview]

---

## 🔥 Top GitHub Repos
| Repo | ⭐ Stars | Lang | Last Updated | Description |
|------|---------|------|-------------|-------------|
...
⚠️ [flag any repo not updated in >12 months]

## 📦 Package Health
| Package | SourceRank | ⭐ Stars | Last Release | Status |
|---------|-----------|---------|-------------|--------|
...
[active = SourceRank ≥20 + recent release; risky = rank <10 or stale]

## 💬 Community Insights
**Trending on HN:**
- [title] — [points] pts · [comments] comments ([date]) — [url]

**Stack Overflow — watch out for:**
- [question title] — [score] votes · [answer_count] answers — [link]

## 🔌 Claude Code Plugins & Skills
| Name | ⭐ | What it adds | Install |
|------|----|-------------|---------|
...

## ⚡ MCP Servers & Connectors
**Runtime-usable:**
| Name | ⭐ Stars | Integrates with | Add command |
|------|---------|----------------|-------------|
...

**Dev-time only (useful in Claude Code sessions, not at runtime):**
- [name] — [what it does during development]

## 📋 Project Templates & CLAUDE.md Patterns
| Repo | ⭐ Stars | Stack | Link | Notable patterns |
|------|---------|-------|------|-----------------|
...

## 🖥️ CLI Tools
| Tool | ⭐ / Downloads | Role | Install |
|------|--------------|------|---------|
...
[runtime / dev-time / build-time tags shown per tool]

---

## Design Decisions
> Specific decisions these results should change about how you build this. Each point names the source (repo, package, SO question, README finding) it came from.

1. **[Concrete decision]** — [why, tied to a specific finding]
2. **[Concrete decision]** — [why, tied to a specific finding]
3. **[Concrete decision]** — [why, tied to a specific finding]
4. **[Gotcha to plan for]** — [source: SO question / README warning / package staleness]
5. **[Best starting point]** — read [repo name]'s [specific file or section] before writing any code; their [pattern] maps directly to your [component]
```
