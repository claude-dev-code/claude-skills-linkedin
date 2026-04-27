# Claude Skills for LinkedIn

A bundle of 4 Claude Code skills that automate LinkedIn outbound, engagement, and enrichment via the [LinkUp API](https://app.linkupapi.com) MCP server.

All skills share the same daily LinkedIn safety caps per account:

| Cap | Limit / day | Tool action it gates |
|---|---|---|
| Invites | **20** | `linkedin_network/invite` |
| Profile gets | **100** | `linkedin_profiles/get` |
| Searches | **15** | `linkedin_profiles/search_*` |
| Comments (feed-engage only) | **15** | `linkedin_content/comment` |

The cap is shared across the 4 skills — hitting it from any one carries over. Each skill checks `linkupapi_get_logs` at Stage 0 before running and trims the campaign size or stops gracefully if the budget is exhausted.

## Skills

### `linkedin-outreach/` — Full B2B outreach campaign
Define an ICP, find target companies, extract decision makers (CEO/CTO/Founder), filter false positives, send connection requests, optionally arm a webhook for accept-triggered messages. 8-stage flow with intent-first ICP discovery (sell to companies / recruit talent / network), draft ICP proposal, batched-form filter editing, multi-account selection with dedup-and-round-robin, and mandatory campaign log persistence.

### `linkedin-high-intent/` — High-intent leads from a post
Given a LinkedIn post URL, scrape commenters and reactors, filter on visible signal (no enrichment), enrich the matches, score against the ICP, and launch a targeted outreach campaign on confirmed matches. Personalized note referencing the post boosts accept rate substantially.

### `linkedin-feed-engage/` — Auto-comment on ICP feed
Scroll the LinkedIn feed, identify posts authored by ICP-matching profiles, generate value-adding 2-3 sentence comments (auto, templated, or HITL-approved), and post them with strict pacing (30-60s between comments, max 15/day). Used as a warm-up before outreach: prospect sees your name on their post, then receives an invite.

### `linkedin-enrich/` — Bulk URL enrichment
Given N LinkedIn profile URLs, return enriched data: current role, company, location, experience, skills, plus professional email when findable. Output as inline markdown table, CSV, or JSON. De-dups against `./enrichments/` from the last 30 days to avoid re-paying for the same `get`.

## Installation

Drop the 4 skill folders into your local Claude Code skills directory:

```bash
git clone <this-repo> /tmp/claude-skills-linkedin
cp -R /tmp/claude-skills-linkedin/linkedin-* ~/.claude/skills/
```

Then restart Claude Code. The skills are auto-discovered and triggered by the user's natural-language prompts (e.g. "outreach campaign", "enrich these profiles", "comment on my feed", "find leads from this post URL").

## Required MCP server

All 4 skills require the `linkupapi` MCP server connected with at least one LinkedIn account in `connected` status. See https://app.linkupapi.com to get your API key and connect a LinkedIn account.

In Claude Code:
```bash
claude mcp add --transport http linkupapi http://YOUR_LINKUP_HOST/mcp \
  --header "x-api-key: YOUR_API_KEY"
```

## Cross-skill data flow

Each skill writes to a shared local directory:
- `./campaigns/{date}-{slug}.md` — outreach + high-intent + feed-engage campaign logs
- `./enrichments/{date}-{slug}.md` — enrichment logs (and the CSV/JSON outputs)

Subsequent runs of any skill **dedupe against these logs** so a profile is never invited / commented on / enriched twice unnecessarily. The campaign log is the source of truth for accept-rate analysis after 7-14 days.

## License

Private use only.
