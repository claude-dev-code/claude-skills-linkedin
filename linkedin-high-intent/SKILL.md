---
name: linkedin-high-intent
description: Find high-intent leads from a LinkedIn post — scrape commenters and reactors, score each profile against an ICP, and launch a targeted outreach campaign on the matches. Use when the user says "high-intent leads", "scrape commenters", "engagers from post", "post reactions outreach", "warm leads from a LinkedIn post", or pastes a LinkedIn post URL and asks to find prospects.
---

# High-Intent Leads from a LinkedIn Post

The premise: people who comment or react on a relevant post are 5-10× more likely to accept a connection or reply than cold outreach. This skill turns a single post URL into a filtered, ICP-matched outreach campaign.

## Required prerequisites
1. `linkupapi` MCP connected with at least one LinkedIn account (status `connected`). Verify with `linkupapi_list_accounts`.
2. Credit balance ≥ 50 (typical run = 30-50 credits).
3. A LinkedIn post URL the user wants to mine.

## Daily LinkedIn safety caps (MANDATORY — enforced before any tool call)

LinkedIn rate-limits per account. This skill respects:
- **20 connection requests / day** (invites)
- **100 profile gets / day** (`linkedin_profiles get`)
- **15 searches / day** (`search_people` / `search_companies`)

**Before Stage 1**, run `linkupapi_get_logs` filtered to the last 24h on the chosen account. Count today's:
- `linkedin_network/invite` calls
- `linkedin_profiles/get` calls
- `linkedin_profiles/search_*` calls

Compute remaining budget per category. **Never override the caps** — trim the run or stop. Surface remaining budget in the Stage 0 echo so the user sees it before confirming.

If `linkupapi_get_logs` is unavailable or empty, fall back to scanning `./campaigns/*.md` and `./enrichments/*.md` for today's date and counting entries.

## Stage 0 — Collect post URL + ICP

Use `AskUserQuestion` (single batched form):

1. **Post URL** — free-text (full LinkedIn post URL, must contain `/posts/` or `/feed/update/`)
2. **ICP** — same shape as `linkedin-outreach` Stage 0: titles, sectors, sizes, geo. If the user has a saved ICP file (`./icp/*.json`), offer to reuse it.
3. **Engagement type**:
   - Commenters only (highest intent — recommended)
   - Reactors only (broader funnel)
   - Both
4. **Volume cap** — max invites this run. **Hard ceiling = today's remaining 20-invite budget**. Default = `min(10, remaining_invites)`.
5. **Note style** — same options as `linkedin-outreach`: no note / personalized / custom. Recommended here: personalized note referencing the post (accept rate is much higher when you reference the engagement).

Echo the brief with budget math:
```
Post:                {url}
ICP:                 {summary}
Engagement:          commenters + reactors
Volume cap:          10 (today: 8 invites used / 12 remaining)
Profile budget:      45 used / 55 remaining today
Note style:          short personalized
Est. credits:        ~35
```
Wait for "yes" before continuing.

## Stage 1 — Pull engagers from the post

Tool: `linkedin_content`. Load schema with `ToolSearch query="select:mcp__linkupapi__linkedin_content"` to confirm action names. Likely:
- `get_post_reactions` for reactors
- `get_post_comments` for commenters

```json
{"account_id": "...", "action": "get_post_reactions", "params": {"post_url": "<url>", "limit": 100}}
```

Run the chosen action(s). Merge results, dedupe by `profile_url`. Drop:
- `LinkedIn Member` anonymized entries (URL contains `headless`)
- The post author themselves
- Profiles with missing `profile_url`

Cost: ~1 credit per 10 engagers. Not counted against the 15-search cap.

## Stage 2 — Pre-filter on visible signal (no enrichment yet)

Each engager comes back with `name`, `headline`, sometimes `current_company`. Score each against the ICP using ONLY visible fields:

- ✅ Clear match — headline mentions target role/industry → enqueue for enrichment
- ❌ Clear mismatch → drop without burning a profile-get credit
- ⚠️ Ambiguous → enqueue if remaining profile-get budget allows

Surface the triage:
```
Engagers found:                47
Clear matches (will enrich):   12
Ambiguous (will enrich):        8
Clear non-matches (dropped):   27
Profile budget:                55 remaining → fits all 20.
```

Stop here and ask if user wants to drop ambiguous to save budget when budget is tight.

## Stage 3 — Enrich matched profiles

Tool: `linkedin_profiles get`, sequential with **1-2s sleep** between calls.

**HARD CAP**: do NOT exceed today's remaining profile-get budget. If 12 strong matches but only 5 budget left, enrich the 5 strongest by visible signal (headline match + current_company match) and write the rest to a "deferred to tomorrow" list in the campaign log.

For each enriched profile, verify CURRENT role (`experience[0]`) actually matches the ICP. Drop people whose recent job changed (false positives).

## Stage 4 — Score and select for outreach

Final scoring per prospect:
- Strong match (current role + industry + size all align) → high priority
- Partial match (2 of 3) → medium
- Weak match (1 of 3) → drop unless volume target is underfilled

Select up to `min(volume_cap, remaining_invite_budget, len(strong_matches + medium_matches))`.

Show the final shortlist to the user (table: name, headline, company, match score, post engagement type) before sending. Wait for "yes".

## Stage 5 — Launch invites (cap-aware)

For each selected prospect:
1. `linkedin_network/check_invitation` → skip if PENDING / CONNECTED
2. Generate the personalized note (if user opted in). Reference the post:
   > "Hi {first_name}, saw your {comment|reaction} on {post_topic_short} — same space we're operating in. Would love to connect."
   Keep ≤250 chars. No links.
3. `linkedin_network/invite` with `identifier=<handle>` and `message=<note>`
4. Sleep 1-2s between invites
5. **STOP** the moment today's 20-invite budget is exhausted

Report partial completion if cap hits before the volume target.

## Stage 6 — Persist & report (mandatory)

Write `./campaigns/{YYYY-MM-DD}-high-intent-{post-slug}.md`:
1. Source post URL + author + topic
2. ICP definition
3. Engagement counts (commenters / reactors / merged / deduped)
4. Triage breakdown (matches / ambiguous / drops)
5. Enrichment results (confirmed / false positives)
6. Invites sent table: `# | Name | Company | Engagement type | Note sent | invitation_urn`
7. Daily caps remaining after this run
8. Deferred prospects list (for tomorrow)

Concise on-screen summary. Include `account_id`, credits consumed, `start_balance → end_balance`.

## De-dup with previous campaigns

**Before Stage 5**, glob `./campaigns/*.md` and `./enrichments/*.md` for any `profile_url` / `invitation_urn` already in the new shortlist. Drop those silently and surface the dedup count in the report.

## Common pitfalls

- **Old posts**: if the post is >30 days old, many commenters have changed jobs. Stage 3 enrichment is critical.
- **Author = ICP**: don't invite the post author themselves.
- **Same-company clustering**: if 5 of 10 matches are from the same target account, surface this — could trigger LinkedIn anti-spam.
- **Note that mentions the post**: the note is the entire reason this skill out-converts cold outreach. Don't default to "no note" here unless user insists.
- **Burning profile budget on non-matches**: Stage 2 visible-signal filter is essential.

## Tool quick reference

| Tool | Action | Cost | Daily cap |
|---|---|---|---|
| `linkedin_content` | `get_post_comments` / `get_post_reactions` | 1/10 | engagement (uncapped) |
| `linkedin_profiles` | `get` | 1 | **100/day shared** |
| `linkedin_network` | `check_invitation` | 1 | (pre-flight, no cap) |
| `linkedin_network` | `invite` | 1 | **20/day** |
| `linkupapi_get_logs` | — | 0 | run at Stage 0 |
| `linkupapi_get_credits` | — | 0 | balance check |
