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

## Phase 2.5 — Community Detection + Blog Discovery (silent, runs in parallel)

Run both steps simultaneously before Phase 3. They set variables used in Phase 3 Sections 3 and 8. Do not surface findings to the user yet — they feed into the search, not the output directly.

---

### Community Detection

The skill defaults to HN + Stack Overflow, which are high-signal for open-source dev tooling but low-signal for enterprise SaaS, admin work, and domain-specific professional platforms. This step detects where the community for *this specific domain* actually lives before running searches.

Run these three queries in parallel:

```bash
# 1. Primary community platform
WebSearch "[USECASE] community forum practitioners ask questions 2025"

# 2. Dedicated Stack Exchange site
WebSearch "[USECASE] stackexchange dedicated community Q&A"

# 3. Subreddit
WebSearch "[USECASE] reddit community subreddit discussion"
```

From the results, identify and record:

**`VENDOR_COMMUNITY`** — if the domain is a commercial platform, it almost certainly has a vendor-hosted forum. Common examples:

| Platform | Community URL |
|----------|--------------|
| Salesforce | trailhead.salesforce.com/trailblazer-community |
| SAP | community.sap.com |
| ServiceNow | community.servicenow.com |
| Atlassian | community.atlassian.com |
| Microsoft 365 | techcommunity.microsoft.com |
| Shopify | community.shopify.com |

If detected, vendor communities are the highest-signal source for that platform — use them as the primary community search target.

**`STACK_EXCHANGE_SITE`** — check whether a dedicated Stack Exchange site exists for this domain. Dedicated sites have far higher signal than the `tagged=` filter on stackoverflow.com because every question is domain-specific. The site ID is the subdomain prefix:

- `salesforce` → salesforce.stackexchange.com
- `dba` → dba.stackexchange.com
- `unix` → unix.stackexchange.com
- `gis` → gis.stackexchange.com

If a dedicated site is found, record its ID and use it in Section 3. If none exists, default to `stackoverflow.com` with the `[TAGS]` filter.

**`SUBREDDIT`** — extract subreddit names from results (e.g. `r/salesforce`, `r/devops`, `r/MachineLearning`). If multiple are found, take the most domain-specific one.

**`HN_RELEVANT`** — set `true` only when the domain is open-source software tooling, developer infrastructure, or a new framework/product likely to have Show HN posts. Set `false` for enterprise SaaS admin work, low-code platforms, or any domain where HN search returned off-topic results during Phase 0 or prior sessions.

---

### Blog Discovery

Run in parallel with Community Detection.

**Step 1 — Find the curated list:**

```bash
WebSearch "top [USECASE] blogs [CURRENT_YEAR]"
```

Identify the top curated list article in the results (e.g. "20 Most Popular Salesforce Blogs 2025", "Best DevOps Blogs 2025"). These lists are maintained by the community and give 10–20 filtered blogs in a single fetch — far more efficient than discovering individual blogs through search.

WebFetch the top curated list URL:
```
WebFetch [top curated list URL]
```

Extract: blog name, URL, specialization. Filter to blogs relevant to the specific domain and use case from Phase 1. Prioritise blogs that cover release notes or breaking changes — these are the most valuable for practitioners keeping up with a platform.

**Step 2 — Direct practitioner search:**

```bash
WebSearch "[USECASE] blog best practices [CURRENT_YEAR]"
```

This surfaces individual posts from smaller blogs that may not appear on curated lists. Extract the domain of each result (not the post URL, the root domain) as a candidate blog.

Record the combined, deduplicated list as **`BLOG_LIST`** — used in Phase 3 Section 8.

---

## Phase 3 — Execution Order

Run the 8 sections below in 5 staged passes. Within each stage, fire all listed sections in parallel. Do not start a stage until the previous stage is complete.

| Stage | Sections | Rationale |
|-------|----------|-----------|
| **1** | §4 Skills + Plugins | Fastest check. Local skills scanned first, then online search always runs regardless of what is found locally. Findings from both are compared in Phase 4 — never skip the online search because a local skill was found. |
| **2** | §1 GitHub Repos | Find the top candidates. Repo names and package names from this stage feed every stage that follows. |
| **3** | §2 Package Health · §9 Platform Release Currency | Both validate and contextualise Stage 2 findings. Package health checks library stability; release currency checks what changed recently and what is currently broken. Neither requires the other — run in parallel. |
| **4** | §5 MCP Servers · §6 Templates · §7 CLI Tools | None of these three inform each other. All benefit from knowing repo/package names from Stage 2. Run in parallel. |
| **5** | §3 Community Insights · §8 Blogs | Search using the general `[USECASE]` topic term — not specific package names. Specific tool signal is already captured in the Issues check (§1) and Release Currency (§9). Community and blog searches serve a different purpose: broad practitioner perspective on whether the overall approach is right, community consensus on the problem space, and discovery of tools not yet found. Run in parallel. |

Variable substitutions used throughout the sections below:
- `[USECASE]` — primary URL-encoded search term derived from Phase 1
- `[USECASE2]` — a variant search term for broader coverage
- `[TAGS]` — comma-separated Stack Overflow tags for this stack (e.g. `python,ffmpeg,flask`)
- `[KEY-PACKAGES]` — the 3–5 most critical packages for this stack — **populated from Stage 2 results before Stages 3–5 run**

---

## Phase 3 — Sections

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

**After reading READMEs, check open Issues on the top 2 repos:**

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/issues?state=open&per_page=20&sort=updated"
```

Search issue titles and bodies for keywords related to the use case (platform names, feature areas, error types). Record internally:
- Any issues describing current breakage or known incompatibility
- Any issues flagging a dependency or API change that broke functionality
- Total open issue count and most recent activity as a health signal

Do not dump all issues into the output — use these notes only for the Known Breakage section in Phase 4.

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

### 3. Community Insights (uses detected platforms from Phase 2.5)

Use the community sources identified in Phase 2.5 rather than defaulting to HN and Stack Overflow. Run all applicable sub-searches in parallel.

**Vendor community** (if `VENDOR_COMMUNITY` was detected):
```bash
WebSearch "[USECASE] site:[VENDOR_COMMUNITY_DOMAIN]"
```
Extract top 3–5 threads. Show: thread title, URL. Flag any thread describing a known issue, limitation, or workaround as a "Watch out for" warning.

**Stack Exchange** (use `STACK_EXCHANGE_SITE` from Phase 2.5):

If a dedicated site was detected (e.g. `salesforce`):
```bash
curl -s "https://api.stackexchange.com/2.3/search?order=desc&sort=votes&intitle=[USECASE]&site=[STACK_EXCHANGE_SITE_ID]&key=$SO_KEY&pagesize=5"
```

If no dedicated site, fall back to stackoverflow.com with tags:
```bash
curl -s "https://api.stackexchange.com/2.3/search?order=desc&sort=votes&intitle=[USECASE]&tagged=[TAGS]&site=stackoverflow&key=$SO_KEY&pagesize=5"
```

Extract: `items[].title`, `items[].score`, `items[].answer_count`, `items[].link`. Show top 3 as "Watch out for" warnings.

**Reddit** (if `SUBREDDIT` was detected):
```bash
WebSearch "[USECASE] reddit r/[SUBREDDIT] discussion"
```
Extract top 3 threads. Show: post title, URL. Reddit surfaces honest practitioner opinions and real-world implementation experiences that rarely appear in official documentation.

**Hacker News** (only if `HN_RELEVANT = true`):
```
WebFetch https://hn.algolia.com/api/v1/search?query=[USECASE]&tags=show_hn&hitsPerPage=8
```
Extract: `hits[].title`, `hits[].points`, `hits[].num_comments`, `hits[].created_at`, `hits[].url`. Show top 3 by points. Skip this entirely if `HN_RELEVANT = false` — do not run the search or show an empty section.

---

### 4. Claude Code Plugins & Skills

**Step 1 — Scan locally installed skills:**

```bash
ls ~/.claude/skills/
```

For each directory found, read its SKILL.md:
```bash
cat ~/.claude/skills/{skill-name}/SKILL.md
```

Extract the `name` and `description` fields from the frontmatter. Assess relevance to the current use case from Phase 1. A skill is relevant if its description covers the same functional area, even partially. Record all relevant local skills — they are used in Phase 4 for comparison against online findings. Do not stop or narrow the search based on what is found here.

**Step 2 — Online search (always runs regardless of Step 1 findings):**

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

### 8. Blogs (uses `BLOG_LIST` from Phase 2.5)

For the top 5 blogs in `BLOG_LIST`, check for recent relevant content:

```bash
WebSearch "[BLOG_NAME] [USECASE]"
```

Or if the blog has a known URL pattern (e.g. `salesforceben.com/salesforce-field-service`), WebFetch it directly.

Extract per blog: name, URL, specialization, most relevant recent post title if found. Flag blogs that cover release notes or breaking changes — these are the highest-value ongoing resource for practitioners keeping up with a platform.

Report in output as a table. Include the curated list source that surfaced them so the user can explore the full list beyond the top 5.

---

### 9. Platform Release Currency

For **each key dependency** identified in Stage 2 — every library, framework, or platform the project relies on — run Steps 1–3 below separately. A project like Nora Job Search has three distinct dependencies to check (JobSpy, Playwright, Playwright-MCP), each with its own release cadence, its own changelog source, and its own breakage profile. Do not run this section once for the project as a whole; run it once per key dependency.

The goal is not just "what's new" — it is "what changed recently that affects this use case, and what is currently broken."

**Step 1 — Determine the current version of each key dependency:**

For versioned platforms with named releases (Salesforce, Shopify, etc.): identify the current release name and date from official docs or community sources.

For libraries and frameworks: check the latest version from the package registry or GitHub releases for each dependency:

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/{owner}/{repo}/releases?per_page=3"
```

Record the current version per dependency. Use these versions throughout the remaining search to tag guidance as current-era vs. potentially outdated — a highly-voted Stack Overflow answer written against a version from two years ago may describe deprecated behaviour.

**Step 2 — Find the best changelog source for each dependency:**

Official changelogs (GitHub releases, CHANGELOG.md, official docs) are the baseline for each. For commercial platforms, also search for the best third-party practitioner aggregator — these writers cover every release with plain-language, practitioner-focused summaries that are more actionable than official release notes:

```bash
WebSearch "[DEPENDENCY NAME] release notes [CURRENT_YEAR] changelog"
WebSearch "best [DEPENDENCY NAME] release coverage blog [CURRENT_YEAR]"
```

Every major platform has an equivalent to corraogroup.com/blog for Salesforce — a small number of practitioners who cover every release in depth. For libraries without formal changelogs (e.g. a scraper library like JobSpy), the GitHub Issues tracker is often the most accurate release notes source — it is where users report when an external platform broke the library. Check it specifically.

**Step 3 — Fetch the last 2–3 releases from the best source found for each dependency:**

WebFetch the changelog or aggregator URL per dependency. Extract and record internally:
- Breaking changes relevant to the use case
- New native capabilities that may replace custom solutions being planned
- Deprecated features the project might accidentally rely on

**Step 4 — For libraries that interact with live external services:**

If a key library scrapes or calls a live platform (e.g. a job scraper calling LinkedIn, an SDK calling a social media API, a scraper calling any regularly-updated website), the live platform itself may have broken the library without any update to the library's release notes. Check specifically:

```bash
WebSearch "[LIBRARY NAME] [PLATFORM NAME] broken [CURRENT_YEAR]"
WebSearch "[LIBRARY NAME] [PLATFORM NAME] not working issue"
```

Also check the library's GitHub Issues (already fetched in Section 1) for open issues mentioning the live platform. This is often the only place this information exists — it won't appear in package health scores or official changelogs.

Record all findings for Phase 4 synthesis and the Known Breakage output section.

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

### Known Breakage Compilation

Collect all breakage signals found across Section 1 (GitHub Issues), Section 9 (Platform Release Currency), and Section 3 (Community Insights). For each signal:
- Name the specific library or platform affected
- Describe what is broken or changed in plain language
- Link to the source (GitHub issue, release note, community thread)

Only include confirmed breakage or instability — not general limitations or design tradeoffs. Those belong in Design Decisions. The Known Breakage section is strictly for things that are currently not working as documented or that changed in a way that will affect the project.

---

### Local vs Online Skills Comparison

If Section 4 found both locally installed skills AND online skills/plugins covering the same functional area, produce a comparison for each overlapping pair:

- **What the local skill does** — based on its SKILL.md description
- **What the online alternative does** — based on registry or repo description
- **Pros of the local skill** — already installed, known behavior, tailored to this project's existing patterns
- **Cons of the local skill** — may be narrower in scope, not updated, or missing capabilities the online version has
- **Recommendation** — which to use and why, or whether combining both makes sense

This comparison goes in the output under the Claude Code Plugins & Skills section. If there is no overlap between local and online findings, omit the comparison entirely.

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
**[Detected community — e.g. Trailblazer Community / r/salesforce / community.sap.com]:**
- [thread title] — [url]

**[Stack Exchange site — e.g. salesforce.stackexchange.com] — watch out for:**
- [question title] — [score] votes · [answer_count] answers — [link]

**Reddit r/[subreddit]** (if detected):
- [post title] — [url]

**Hacker News** (only shown if HN_RELEVANT = true):
- [title] — [points] pts · [comments] comments ([date]) — [url]

## ⚠️ Known Breakage
> What is currently broken, unstable, or recently changed in this stack. Sources: GitHub Issues on top repos, Platform Release Currency findings, recent community threads.

- **[Library or platform]**: [what is broken or changed] — [source URL]
...
[Omit this section entirely if no known breakage or instability was found.]

## 📰 Blogs
| Blog | Specializes in | Covers releases? | URL |
|------|---------------|-----------------|-----|
...
> Full curated list sourced from: [curated list URL]

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
