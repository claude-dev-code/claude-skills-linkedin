# LinkupAPI — Claude Skills

> **Connect your AI Agent to B2B Channels.**
> [LinkupAPI](https://app.linkupapi.com) is a unified API that lets agents act on the channels where B2B revenue actually happens. Today: full LinkedIn coverage (outbound, engagement, content, recruiter, enrichment, webhooks). More channels are landing on the same action-based interface.

This repo packages 4 Claude Code skills that turn LinkupAPI's capabilities into ready-to-run, auditable workflows. They're the fastest way to put an AI agent on a real LinkedIn account without writing glue code.

## Skills

### `linkedin-outreach/` — Full B2B outreach campaign
Define an ICP, find target companies, extract decision makers (CEO/CTO/Founder), filter false positives, send connection requests, optionally arm a webhook for accept-triggered messages. 8-stage flow with intent-first ICP discovery (sell / recruit / network), draft ICP proposal, batched-form filter editing, multi-account selection with dedup-and-round-robin, and mandatory campaign log persistence.

### `linkedin-high-intent/` — High-intent leads from a post
Given a LinkedIn post URL, scrape commenters and reactors, filter on visible signal (no enrichment), enrich the matches, score against the ICP, and launch a targeted outreach campaign on confirmed matches. The "saw your comment on…" note boosts accept rate substantially over cold.

### `linkedin-feed-engage/` — Auto-comment on ICP feed
Scroll the feed, identify posts authored by ICP-matching profiles, generate value-adding 2-3 sentence comments (auto, templated, or HITL-approved), and post them with strict pacing (30-60s between, max 15/day). Used as a warm-up before outreach: prospect sees your name on their post, then receives an invite.

### `linkedin-enrich/` — Bulk URL enrichment
Given N LinkedIn profile URLs, return enriched data: current role, company, location, experience, skills, plus professional email when findable. Output as inline markdown table, CSV, or JSON. De-dups against `./enrichments/` from the last 30 days so you never re-pay for the same lookup.

## Daily safety caps (shared across all 4 skills)

Each skill is rate-limit-aware and respects these per-account daily caps before any tool call:

| Cap | Limit / day | Tool action it gates |
|---|---|---|
| Connection requests | **20** | `linkedin_network/invite` |
| Profile gets | **100** | `linkedin_profiles/get` |
| Searches | **15** | `linkedin_profiles/search_*` |
| Comments (feed-engage only) | **15** | `linkedin_content/comment` |

The cap is **shared across all 4 skills** — hitting it from any one carries over. Each skill queries `linkupapi_get_logs` at Stage 0 to compute remaining budget, then trims the campaign size or stops gracefully if the budget is exhausted. Same-day duplicate work is deduped via `./campaigns/*.md` and `./enrichments/*.md` logs.

## Installation

```bash
git clone https://github.com/claude-dev-code/claude-skills-linkedin /tmp/claude-skills-linkedin
cp -R /tmp/claude-skills-linkedin/linkedin-* ~/.claude/skills/
```

Restart Claude Code. The skills are auto-discovered and triggered by natural-language prompts: "outreach campaign", "enrich these profiles", "comment on my feed", "find leads from this post URL", etc.

## Connect the LinkupAPI MCP server

All 4 skills require the LinkupAPI MCP server with at least one channel account in `connected` status. Sign up at [app.linkupapi.com](https://app.linkupapi.com) to get your API key and connect an account (LinkedIn today; more channels rolling out).

In Claude Code:
```bash
claude mcp add --transport http linkupapi https://mcp.linkupapi.com/mcp \
  --header "x-api-key: YOUR_API_KEY"
```

The remote MCP server runs OAuth 2.1 + HTTP streamable transport. After registration, Claude Code redirects you to the LinkupAPI dashboard for one-click consent.

## Cross-skill data flow

Each skill writes to a shared local directory inside the project where Claude Code runs:
- `./campaigns/{date}-{slug}.md` — outreach + high-intent + feed-engage campaign logs
- `./enrichments/{date}-{slug}.md` — enrichment logs (and the CSV/JSON outputs alongside)

Subsequent runs of any skill **dedupe against these logs**, so a profile is never invited, commented on, or enriched twice unnecessarily. The logs are the source of truth for accept-rate analysis 7-14 days after a campaign.

## Beyond LinkedIn

LinkupAPI is built around an **action-based, multi-channel** interface — the same `{action, params}` shape that powers `linkedin_profiles` / `linkedin_network` / `linkedin_content` will extend to other B2B channels as they land. The skills in this repo are LinkedIn-shaped today; the LinkupAPI surface itself is broader.

## License

Private use only (for now).
