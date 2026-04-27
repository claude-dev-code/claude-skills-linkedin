---
name: linkedin-enrich
description: Enrich a list of LinkedIn profile URLs — get full profile info (current role, experience, skills) plus professional email when findable. Use when the user says "enrich profiles", "find emails from LinkedIn URLs", "bulk enrich", "lead enrichment", "LinkedIn to email", "scrape these profiles", or pastes a list of LinkedIn URLs.
---

# Bulk LinkedIn URL Enrichment

Given N LinkedIn profile URLs, this skill returns enriched data: current role, company, location, experience timeline, skills, and email when findable. Output as a markdown table, CSV, or JSON.

## Required prerequisites
1. `linkupapi` MCP connected with at least one LinkedIn account (status `connected`).
2. Credits ≥ `2 × number_of_urls` (1 credit per profile + 1 per email lookup, plus optional validation).
3. A list of LinkedIn URLs from the user (inline, file path, or copy-paste).

## Daily LinkedIn safety caps (MANDATORY — enforced)

- **100 profile gets / day** — hard ceiling for this skill
- 15 searches/day — not relevant (no search calls)
- 20 invites/day — not relevant (no invites)

**Before Stage 1**, run `linkupapi_get_logs` for last 24h on the chosen account, count today's `linkedin_profiles/get`. The skill enriches only `min(N, 100 - already_used_today)`. If user provides 50 URLs but 80 profile gets are already used today, enrich the 20 most-prioritized and defer the rest.

If `linkupapi_get_logs` is unavailable, fall back to scanning today's `./campaigns/*.md` and `./enrichments/*.md` for `linkedin_profiles/get` evidence and counting.

## Stage 0 — Collect input

`AskUserQuestion`:

1. **URL source**:
   - Paste list inline (one URL per line)
   - Path to a file (`.csv` / `.txt` / `.md` — agent reads via `Read`)
2. **Email enrichment**: yes / no (each adds 1 credit per profile, plus validation if enabled)
3. **Validate emails**: yes / no (recommended yes for paid lists; +1 credit per validated email)
4. **Output format**:
   - Markdown table inline (default for ≤50 URLs — readable in chat)
   - CSV file → `./enrichments/{date}-{slug}.csv`
   - JSON file → `./enrichments/{date}-{slug}.json`
5. **Output fields** — checkboxes:
   - Always: name, current title, current company, location, headline
   - Optional: industry, full experience timeline, skills, education, profile_picture_url
   - If email enabled: email, validation status

Echo the plan with credit + budget math:
```
URLs provided:           50
URLs after dedupe:       47
Profile budget today:    100 used / 100 cap → 0 remaining 🚫 STOP
                                  OR
Profile budget today:    20 used / 80 remaining → enrich up to 47 ✓

Estimated credits:       47 (profiles) + 47 (emails) + 47 (validate) = 141
Output format:           CSV → ./enrichments/2026-04-27-leads-batch-1.csv
```

Wait for "yes" before continuing.

## Stage 1 — Parse and dedupe URLs

Extract the LinkedIn handle from each URL (`/in/<handle>` or `/in/<handle>/`). Drop:
- Duplicates (same handle)
- `/company/` URLs (this skill is profiles only — surface them in a separate "skipped: company URLs" list)
- `LinkedIn Member` anonymized URLs (URL contains `headless`) — cannot be enriched
- URL-encoded paths — extract raw handle (decode percent-encoding)

Show the user the parsed count vs original. If >20% were dropped, ask for confirmation before continuing (likely a bad input list).

## Stage 2 — Profile enrichment

Tool: `linkedin_profiles get` with `identifier=<handle>`, sequential with **2-3s sleep** between calls (longer than outreach skill — bulk enrichment patterns are more closely watched by LinkedIn).

**HARD CAP** at remaining 100/day budget. If reached mid-run, save partial results, mark remaining as "deferred to tomorrow", and stop gracefully.

For each profile, extract per the user's chosen output fields. Always capture:
- `first_name`, `last_name`, `headline`
- `experience[0].company` → `current_company`
- `experience[0].title` → `current_title`
- `experience[0].company_url` → derive `company_domain` for email lookup (extract last path segment of LinkedIn company URL → match against known domain registry, OR pass the company name to email finder)
- `location`, `industry`

Mark a profile as `enrichment_failed` (not budget-exhausted) if the API returns 404 / private / member-removed. Don't retry.

## Stage 3 — Email enrichment (if requested)

Tool: `linkupapi_enrich` action `find_email`.

**Mode A — by LinkedIn URL (cleanest)**:
```json
{"action": "find_email", "params": {"linkedin_url": "<full url>"}}
```

**Mode B — by name + company domain (fallback when Mode A returns nothing)**:
```json
{"action": "find_email", "params": {
  "first_name": "...", "last_name": "...",
  "company_domain": "<domain>"
}}
```

Cost: 1 credit per call. No LinkedIn-side cap (it's not a LinkedIn API call), but enforce sane volume — stop if more than 200 in a single run unless user explicitly approves.

For each profile that returns an email, run `linkupapi_enrich validate_email` if user opted in (catches typos and dead inboxes). 1 credit each. Skip validation on profiles where `find_email` returned nothing.

## Stage 4 — Output

Build the chosen output format.

**Markdown table** (≤50 URLs, default):
```markdown
| # | Name | Title | Company | Location | Email | Valid |
|---|------|-------|---------|----------|-------|-------|
| 1 | Oleksii Kratko | Founder/CEO | Snov.io | Toronto | oleksii@snov.io | ✓ |
```

**CSV** (>50 or chosen): write to `./enrichments/{date}-{slug}.csv` with header row, one row per profile (including failed enrichments with reason).

**JSON**: array of objects with all enriched fields, suitable for re-import into other workflows.

Show the user the output path + a preview of the first 5 rows. For inline markdown, render the table directly.

## Stage 5 — Persist & report (mandatory)

Always write a log regardless of output format:

`./enrichments/{YYYY-MM-DD}-{slug}.md`:
1. Input source (file path or "inline N URLs")
2. URL counts: provided / deduped / company URLs separated / anonymous dropped / enriched / failed / deferred
3. Email match rate (X / N profiles got an email)
4. Email validation rate (Y / X validated)
5. Failed enrichments table: `url | reason` (private / 404 / removed)
6. Deferred URLs table (for tomorrow's run)
7. Daily caps remaining
8. Output file path (CSV / JSON if produced)

The log lets the user re-run failed/deferred ones tomorrow without losing track.

## De-dup with previous enrichments

**Before Stage 2**, glob `./enrichments/*.md` and `./enrichments/*.json` for any `profile_url` already enriched in the last 30 days. Reuse the cached enrichment instead of re-paying for the `get` call. Surface the dedup count and savings in the Stage 0 echo.

This is the single biggest cost saver — repeat enrichments are a waste.

## Common pitfalls

- **Anonymized URLs**: `/in/headless?...` can't be enriched. Filter at Stage 1.
- **URL-encoded handles** (Greek/Cyrillic etc.): pass the decoded handle to `identifier`, not the URL.
- **Email find rate**: realistic = 40-60% find rate (the rest don't have a discoverable professional email). Don't promise 100%.
- **Daily cap mid-run**: if email is requested, account for BOTH profile + email budget at Stage 0. Better to enrich 80 fully than start 100 and hit cap mid-email-pass.
- **Reusing enrichment for outreach**: this skill's output feeds directly into `linkedin-outreach` Stage 4 verification. The outreach skill should check `./enrichments/` first to avoid re-paying for the same `get`.
- **Personal Gmail returned**: `find_email` sometimes returns personal email (e.g. `firstname@gmail.com`) instead of professional. Surface both when both exist and let the user choose downstream.

## Tool quick reference

| Tool | Action | Cost | Daily cap |
|---|---|---|---|
| `linkedin_profiles` | `get` | 1 | **100/day shared** |
| `linkupapi_enrich` | `find_email` | 1 | — |
| `linkupapi_enrich` | `validate_email` | 1 | — |
| `linkupapi_enrich` | `reverse_email` | 1 | (not used here) |
| `linkupapi_get_logs` | — | 0 | run at Stage 0 |
| `linkupapi_get_credits` | — | 0 | balance check |
