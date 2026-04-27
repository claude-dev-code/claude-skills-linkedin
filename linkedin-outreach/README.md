# Claude Code Skill — LinkedIn Outreach Campaigns

> Run end-to-end LinkedIn outreach campaigns from Claude Code. Define your ICP in a single form, find the right companies, extract real decision makers (not ex-employees), send rate-limited connection requests, and auto-follow-up via webhooks.

[← Back to all Claude LinkedIn Skills](../README.md)

## What it does

This is a **Claude Code skill** that turns the [LinkUp API](https://linkupapi.com) MCP into a guided LinkedIn prospecting workflow. When you type `/linkedin-outreach`, Claude:

1. Pops a structured ICP form (theme, persona, geo, company size, volume, note style, follow-up).
2. Searches LinkedIn for matching companies (`linkedin_profiles.search_companies`, all 4 V2 filters as JSON arrays).
3. Filters out staffing agencies and bad-fit companies.
4. Extracts decision makers per company (`linkedin_profiles.search_people`, all 13 filters available — `network: "S"` for 2nd-degree boost).
5. **Enriches every profile** to drop ex-employees that LinkedIn returns as false positives — the #1 trap of cold LinkedIn search.
6. Pre-flight checks each invitation status before sending.
7. Sends rate-limited connection requests (1–2 sec sleep between, ≤ 80/session).
8. Sets up an `accepted_invitation` webhook so accepts trigger automated follow-up DMs.
9. Reports the campaign with credit budget consumed.

## Who it's for

- **Founders & solo SDRs** running outbound on LinkedIn without a 6-tool sales stack.
- **Recruiters** targeting decision makers at competitor companies (`past_company` filter).
- **Growth teams** wanting reproducible, auditable LinkedIn campaigns rather than untraceable click-and-pray automation.
- **Anyone using Claude Code** who wants their outreach to live next to their code, not in a separate SaaS.

## What you get vs. raw API calls

| Without this skill | With this skill |
|---|---|
| You write the JSON for each MCP call manually | Claude orchestrates the full pipeline |
| You forget which V2 filter is `keyword` vs `title` | The skill enforces `title` for personas |
| ~30–50% of `search_people` results are ex-employees → wasted credits | Stage 4 auto-filters them by re-checking `experience[0]` |
| You re-invite people already pending → wasted credits | Pre-flight `check_invitation` skips them |
| You send invites in parallel → LinkedIn rate-limits you | Sequential loop with sleep enforced |
| You forget to track accepts | Webhook auto-armed for `accepted_invitation` events |

## Install

### One-shot installer

```bash
curl -fsSL https://raw.githubusercontent.com/claude-dev-code/claude-linkedin-skills/main/install.sh | bash
```

### Manual install

```bash
git clone https://github.com/claude-dev-code/claude-linkedin-skills.git
cp -r claude-linkedin-skills/linkedin-outreach ~/.claude/skills/
```

Then restart Claude Code.

## Prerequisites

The `linkupapi` MCP server must be connected:

```bash
claude mcp add --transport sse linkupapi https://mcp.linkupapi.com/sse \
  --header "x-api-key: YOUR_LINKUP_API_KEY"
```

Get your API key at [app.linkupapi.com](https://app.linkupapi.com). At least one LinkedIn account must be connected (the skill will guide you through `linkupapi_login` if not).

## Run it

```
/linkedin-outreach
```

or in plain English:

```
> I want to run a LinkedIn outreach campaign — find AI recruitment SaaS companies in France, target their CEOs and CTOs, send 50 connection requests with auto follow-up.
```

## Credit budget

A standard 50-prospect campaign typically consumes **110–150 credits**:

| Stage | Cost |
|---|---|
| `search_companies` (limit=25) | ~3 credits |
| `search_people` × 12–18 companies | ~12–18 credits |
| `linkedin_profiles.get` × candidates | ~40–60 credits |
| `check_invitation` × confirmed targets | ~30 credits |
| `invite` × confirmed targets | ~30 credits |
| `list_sent` verification | 1 credit |
| **Total** | **~110–150** |

Hard limit is enforced at **80 invites per session** to stay under LinkedIn's weekly cap of ~100 connection requests per account.

## Full playbook

The 8-stage operational playbook (with all V2 params, JSON examples, message templates, and common pitfalls) lives in [SKILL.md](./SKILL.md) — that's the file Claude Code loads.

## Related skills

- `linkedin-engagement` — warm-up campaign (likes/comments on ICP posts before invite) — *coming soon*
- `email-finder-bulk` — parallel email-channel outreach with `linkupapi_enrich.find_email` — *coming soon*
- `recruiting-pipeline` — full recruiter funnel (search jobs → get_candidates → enrich → outreach) — *coming soon*

[← All Claude LinkedIn Skills](../README.md)

## License

MIT — see [LICENSE](../LICENSE).
