# /project-scout

A Claude Code skill that discovers the best repos, packages, MCP servers, Claude plugins, templates, CLI tools, and community resources before you start a new project — tailored to the specific domain and runtime architecture you describe.

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
Pre-fills the first question with your use-case hint — confirms or expands before searching.

## What it does

| Phase | What happens |
|-------|-------------|
| **Phase 0 — Sync** | Silently checks for new official Anthropic skills; installs any missing ones and notifies you |
| **Phase 1 — Interview** | Asks 5 questions: goal, audience, success criteria, context, required capabilities. Derives technical search terms from your answers — does not ask you for them |
| **Phase 2 — Credentials** | Loads GitHub, Libraries.io, and Stack Overflow API keys from `~/.claude/` |
| **Phase 2.5 — Community Detection** | Detects where the domain community actually lives (vendor forum, dedicated Stack Exchange site, subreddit) before running searches |
| **Phase 2.5 — Blog Discovery** | Fetches the top curated blog list for the domain; supplements with a direct practitioner search |
| **Phase 3 — 8-section search** | Runs in 5 staged passes: Skills/Plugins → GitHub Repos → Package Health → MCP Servers + Templates + CLI Tools → Community Insights + Blogs |
| **Phase 4 — Synthesis** | Applicability filter (runtime vs dev-time), local vs online skills comparison, README pattern extraction, design decisions tied to specific findings |

## Output sections

- **Top GitHub Repos** — exact star counts via authenticated API, staleness flags, README pattern notes
- **Package Health** — PyPI / npm / Libraries.io SourceRank, last release date, risk flags
- **Community Insights** — vendor forum, Stack Exchange, Reddit, Hacker News (only when relevant)
- **Blogs** — top 5 from curated list, specialization, whether they cover release notes
- **Claude Code Plugins & Skills** — local installed vs online alternatives, side-by-side comparison when both cover the same area
- **MCP Servers** — tagged runtime-usable vs dev-time only
- **Project Templates** — repos with notable CLAUDE.md patterns worth copying
- **CLI Tools** — tagged build-time / dev-time / runtime
- **Design Decisions** — specific decisions tied to named findings (repo, package, Stack Overflow question, README gotcha)

## API keys used

| File | Used for |
|------|---------|
| `~/.claude/github-token` | Authenticated GitHub search (exact star counts, no rate limit) |
| `~/.claude/libraries-io-key` | Libraries.io SourceRank health scores |
| `~/.claude/stackoverflow-key` | Stack Exchange API |

Any missing key falls back gracefully — the corresponding source is skipped silently.

## Release notes

### v2.0 — 2026-06-01

Major rewrite. The v1.0 GitHub version ran a single-pass parallel search. v2.0 adds a full community-aware, staged search pipeline.

**Added:**

- **Phase 2.5 Community Detection** — three parallel queries identify whether the domain has a vendor community (Salesforce, SAP, Atlassian, etc.), a dedicated Stack Exchange site, or a primary subreddit before any Phase 3 searches run. Sets `VENDOR_COMMUNITY`, `STACK_EXCHANGE_SITE`, `SUBREDDIT`, and `HN_RELEVANT` variables used throughout Phase 3.
- **Phase 2.5 Blog Discovery** — fetches the top curated blog list for the domain and supplements with a direct practitioner search. Produces `BLOG_LIST` used in Phase 3 Section 8.
- **Phase 3 restructured into 5 staged passes** — Skills/Plugins first as an early-exit gate, then GitHub repos, then package health validation, then MCP/templates/CLI tools in parallel, then community insights and blogs using specific package names found in earlier stages.
- **Phase 3 Section 3 — Community Insights** — uses the detected community platforms rather than always defaulting to HN + Stack Overflow. HN is only queried when `HN_RELEVANT = true`.
- **Phase 3 Section 8 — Blogs** — queries top 5 blogs from `BLOG_LIST` for recent relevant content; flags blogs covering release notes or breaking changes as the highest-value ongoing resources.
- **Phase 4 Applicability Filter** — every MCP server, plugin, and CLI tool is tagged `dev-time` or `runtime`. Dev-time tools are separated into a distinct note; only runtime-callable tools appear in the main MCP section.
- **Phase 4 Local vs Online Skills Comparison** — when a locally installed skill and an online plugin cover the same functional area, produces a side-by-side pros/cons comparison and a recommendation.
- **Phase 4 README Pattern Extraction** — architecture decisions, key dependencies, gotchas, and performance warnings from the top 2 repo READMEs are surfaced as named sources in the Design Decisions section.

**Fixed:**

- **Reddit community search** — replaced `site:reddit.com/r [USECASE] community` with `[USECASE] reddit community subreddit discussion`. The `site:` operator is unreliable in the WebSearch tool and returned empty results in practice.
- **Stack Exchange detection** — replaced `[USECASE] site:stackexchange.com` with `[USECASE] stackexchange dedicated community Q&A` for the same reason.
- **Reddit Phase 3 Section 3 search** — replaced `site:reddit.com/r/[SUBREDDIT] [USECASE]` with `[USECASE] reddit r/[SUBREDDIT] discussion`.

### v1.0 — initial release

- 5-question qualification interview
- Single-pass parallel search: GitHub repos, Claude skills/plugins, MCP servers, project templates, CLI tools
- Authenticated GitHub API for exact star counts
- Libraries.io SourceRank package health
- Hacker News Show HN search
- Ranked output table per category with install commands
