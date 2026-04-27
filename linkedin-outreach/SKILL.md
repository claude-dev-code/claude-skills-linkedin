---
name: linkedin-outreach
description: Run a full LinkedIn outreach campaign via the LinkupAPI MCP — define an ICP (Ideal Customer Profile), find target companies, extract real decision makers (CEO/CTO/Founder), filter false positives, send connection requests, and set up automated follow-up via webhooks. Use when the user says "outreach campaign", "prospect on LinkedIn", "find decision makers", "send connection requests", "build a target list", "campaign LinkedIn", "lead generation", "ICP", "target CEO/CTO", or describes a B2B outreach goal. Requires the `linkupapi` MCP server to be connected with at least one LinkedIn account (status `connected`).
---

# LinkedIn Outreach Campaign — LinkupAPI V2

Use this skill to run cold-outreach campaigns on LinkedIn via the `linkupapi` MCP. It codifies the full prospecting → connect → follow-up playbook in 7 stages, with credit budgeting and false-positive filtering built in.

## Required prerequisites — check before you start

1. The user must have the `linkupapi` MCP connected. Verify with `linkupapi_list_accounts`.
   - If the list is **empty**, tell the user they have two options to connect a LinkedIn account, then stop and wait:
     - **Hosted UI** — open https://app.linkupapi.com/account-connection (fastest, handles checkpoints in-browser).
     - **MCP login** — run `linkupapi_login` directly (platform=linkedin, with email+password OR a `login_token`). On `checkpoint_required` → run `linkupapi_checkpoint`.
2. Confirm credit balance with `linkupapi_get_credits`. A 50-prospect campaign typically costs **80–120 credits**. Refuse to start if balance < expected cost; show the user the math.
3. If a webhook for the chosen account(s) doesn't exist with the `accepted_invitation` event, propose to create one (`linkupapi_create_webhook`) so accepts trigger automated follow-up.

## Daily LinkedIn safety caps (MANDATORY — enforced before Stage 1)

Per LinkedIn account, this skill respects:
- **20 connection requests / day** (invites)
- **100 profile gets / day** (`linkedin_profiles get`)
- **15 searches / day** (`search_people` / `search_companies`)

These are **shared budgets across ALL linkedin-* skills** (linkedin-outreach, linkedin-high-intent, linkedin-feed-engage, linkedin-enrich). Hitting the cap from any one skill carries over.

**Before Stage 1**, run `linkupapi_get_logs` filtered to the last 24h on the chosen account. Count today's:
- `linkedin_network/invite` calls → invites used today
- `linkedin_profiles/get` calls → profile gets used today
- `linkedin_profiles/search_*` calls → searches used today

Compute remaining budget per category. Surface in the Stage 0 echo:
```
Daily caps remaining today (account {email}):
  Invites:  X / 20  (Y used today)
  Profiles: X / 100 (Y used today)
  Searches: X / 15  (Y used today)
```

**Hard rules**:
- If `Volume cap > remaining_invites` → trim Volume cap to `remaining_invites` (never override).
- If `searches_needed > remaining_searches` (Stage 1 + Stage 3 combined) → reduce Stage 1 `limit` and number of `search_people` calls until it fits, or stop and ask the user to run tomorrow.
- If `profile_gets_needed > remaining_profiles` (Stage 4) → enrich the highest-priority candidates and write the rest to a "deferred" list in the campaign log.

If `linkupapi_get_logs` is unavailable or empty, fall back to scanning today's `./campaigns/*.md` and `./enrichments/*.md` for evidence of API calls and counting.

## Stage 0 — Discover intent, then derive the ICP, then validate

**Don't run any MCP tool yet.** Stage 0 has three steps and is non-negotiable — it's what protects the user from a wasted 100-credit run on the wrong audience.

The skill does NOT start with company filters. It starts with **why** the user is doing outreach. The flow branches from there: a recruiter targets people directly, a salesperson targets companies first then decision makers, a networker targets people by topic/role.

### Step 0.1 — Ask intent (single question)

Use `AskUserQuestion` with exactly this question (don't bundle other questions yet — the answer drives every subsequent option list):

```yaml
questions:
  - header: "Goal"
    question: "What's the goal of this outreach campaign?"
    multiSelect: false
    options:
      - { label: "Sell to companies",    description: "B2B sales — find companies that match an ICP, then reach their decision makers (CEO/CTO/Head of …)" }
      - { label: "Recruit talent",       description: "Hiring — find individual candidates by role / skill / seniority, no company-level search" }
      - { label: "Network / partnerships", description: "Connect with peers, investors, partners, or thought leaders by topic / role / community" }
      - { label: "Custom (I'll explain)", description: "Free-text — describe the goal in one sentence" }
```

If `AskUserQuestion` is unavailable, ask in plain text: *"Quick check before I build the campaign — what are you trying to do? (1) sell to companies, (2) recruit talent, (3) network/partnerships, (4) something else."*

### Step 0.2 — Derive a draft ICP from the intent

Based on the chosen goal, **propose** a draft ICP in plain text (don't ask yet — show what you'd build). Pick reasonable defaults from the user's product/repo context if available, otherwise leave placeholders.

| Goal | Flow used | What you propose |
|---|---|---|
| **Sell to companies** | Stage 1 → 2 → 3 → 4 → 6 | Theme/keywords, target sector, target company size, target geo, decision-maker titles |
| **Recruit talent** | **Skip Stage 1–2.** Go directly to Stage 3 (`search_people`) | Role title(s), seniority level, skills/keywords, location, optionally `past_company` (poach from competitor) or `school_url` (alumni) |
| **Network / partnerships** | **Skip Stage 1–2.** Go directly to Stage 3 (`search_people`) | Topic/keyword, role title(s), location, optionally `follower_of` or `connection_of` for warm paths |
| **Custom** | Ask the user to describe in one sentence, then map to one of the three flows above | — |

Render the draft as a clean block, e.g. for "Sell to companies":

```
Here's the ICP I'd build — tell me what to change:
  Goal:           Sell to companies
  Theme:          AI recruitment / RecTech
  Target sectors: Software Development, Human Resources Services
  Company size:   11-50, 51-200
  Geo:            France
  Personas:       CEO / Founder, CTO / VP Engineering
  Volume:         25 invites (warm-up)
  Note:           No note (best accept rate on cold C-level)
  Message after accept: Auto-generate a short personalized message per connection
```

### Step 0.3 — Let the user edit the filters (batched form)

Now use `AskUserQuestion` with the **filter set that matches the chosen flow**. Pre-select sensible defaults from Step 0.2 by listing them first / marking `(Recommended)`. The form tool caps at 4 questions per call, so split into 2 batches if needed.

**For "Sell to companies"** — ask the company-side filters first, then persona + tactics:

Round A (audience): Theme · Sectors · Company size · Geo
Round B (execution): Persona titles · Volume · Note style · Message-after-accept

**For "Recruit talent"** — skip company filters entirely:

Round A (candidate): Role titles · Seniority · Skills/keywords · Location
Round B (execution): Volume · Note style · Message-after-accept · Sourcing twist (`past_company` / `school_url` / none)

**For "Network / partnerships"** — also skip company filters:

Round A (audience): Topic/keyword · Role titles · Location · Warm-path twist (`follower_of` / `connection_of` / none)
Round B (execution): Volume · Note style · Message-after-accept

### Step 0.4 — Pick the sending account(s)

After the ICP is locked, run `linkupapi_list_accounts` and present the accounts to the user via `AskUserQuestion` with `multiSelect: true`. Each option label is the account's display name; description shows status + email/handle. Only show accounts with `status = connected`.

```yaml
question: "Which LinkedIn account(s) should send the invites?"
multiSelect: true
options:
  - { label: "<account name 1>", description: "<email> — connected" }
  - { label: "<account name 2>", description: "<email> — connected" }
  ...
```

If the user picks **multiple** accounts, the campaign is split across them and **the same person must NEVER be invited from two different accounts** (LinkedIn flags duplicate invites and accept rate collapses). Enforce this:
- Pool all candidate profiles after Stage 4.
- Deduplicate by `profile_url`.
- Round-robin assign each unique candidate to exactly one sending account.
- Cap each account at the per-session/weekly safe limits independently (80/session, 100/rolling week per account).
- In Stage 6, run the `check_invitation` and `invite` calls **with the assigned account_id** for each candidate.

After this step, **echo the final ICP + account selection back in 4–6 lines** and ask "Should I proceed?" (yes/no). Do NOT touch any further MCP tool until the user confirms.

### How the answers map to the next stages

| Goal | Stage 1 (companies) | Stage 3 (people) | Notes |
|---|---|---|---|
| Sell to companies | runs with `keyword`+`sector`+`location`+`company_size` | runs with `company_url` (from Stage 1 results) + `title` | Full 7-stage flow |
| Recruit talent | **skipped** | runs with `title`+`location`+`keyword` (skills) + optional `past_company` / `school_url` | Stage 4 still runs (verify current role) |
| Network / partnerships | **skipped** | runs with `title`+`location`+`keyword` + optional `follower_of` / `connection_of` | Stage 4 still runs |

| Generic answer | Used in |
|---|---|
| Volume | Hard cap on Stage 6 invite count, also informs `limit` in Stage 1/3 (Volume × 2.5 ≈ candidates needed since some are filtered) |
| Note style | Stage 6 `invite.params.message` |
| Message-after-accept | Stage 7 webhook + send-message-on-accept logic (auto-personalized / custom template / none) |

## Stage 1 — Find target companies

Tool: `linkedin_profiles` action `search_companies`.

The V2 exposes exactly **4 filters** on companies (this is the full list — there's no `industry` filter on companies, use `sector`). **Pass values as JSON arrays** for OR-stacking — that's the V2-native syntax. Plain strings are also accepted (single value).

| Param | Type | Purpose |
|---|---|---|
| `keyword` | array of strings | Free-text match (name, description, tagline). Example: `["AI recruitment", "applicant tracking"]` |
| `sector` | array of strings | LinkedIn industry of the company. Example: `["Software Development", "Internet", "Computer Software"]` |
| `location` | array of strings | Geo of the HQ / branch. Example: `["France", "United Kingdom"]` |
| `company_size` | array of strings | Employee bracket. Allowed values: `1-10`, `11-50`, `51-200`, `201-500`, `501-1000`, `1001-5000`, `5001+` |

**Filter combination semantics:**
- Multiple values inside one array → **OR** (`["AI recruitment", "ATS"]` = either match)
- Different fields → **AND** (`sector=[...]` AND `company_size=[...]` AND …)

```json
{
  "account_id": "<from list_accounts>",
  "action": "search_companies",
  "params": {
    "keyword": ["AI recruitment", "recruiting copilot", "AI sourcing"],
    "sector": ["Software Development", "Human Resources Services"],
    "location": ["France", "United Kingdom"],
    "company_size": ["11-50", "51-200"],
    "limit": 25
  }
}
```

- Cost: 1 credit per 10 results (so `limit=25` → 3 credits).
- Pagination: `offset` + `limit` (preferred) or `start_page` + `end_page`.
- The result objects come back with `industry` and `location` split out (V2 post-processing splits `"industry • city"` into separate fields), so filtering in Stage 2 is straightforward.
- Save the raw list, then in Stage 2 manually filter out **non-fits** (staffing agencies dressed as tech, pure consultancies, etc.).

## Stage 2 — Filter companies (LLM judgment)

Look at each result's `industry`, `name`, `location`, employee count. **Drop**:
- Companies whose industry is `Staffing and Recruiting` if the user wants RecTech vendors (not agencies).
- Companies whose name suggests they are a service buyer of the user's product, not a fit.
- Duplicates (same root domain) and obvious test/personal pages.

Show the filtered shortlist to the user before continuing if filtering is aggressive (>50% dropped). Otherwise continue.

## Stage 3 — Extract decision makers per company

Tool: `linkedin_profiles` action `search_people`.

**Use `title`, NOT `keyword`, to filter on roles.** `keyword` matches anywhere in the profile (headline, posts, summary) → too noisy. `title` matches the current job title only → precise.

The V2 exposes **13 filters** on people search. **Pass values as JSON arrays** for OR-stacking; different fields are AND-combined. (Plain strings are accepted for single values too.)

| Param | Type | Purpose | Example |
|---|---|---|---|
| `title` | array | **Current job title** — use this for personas | `["CEO", "CTO", "Co-Founder", "Founder", "Chief Executive Officer"]` |
| `company_url` | array | Filter to current employees of one or more companies | `["https://www.linkedin.com/company/talentify-io/"]` |
| `company_name` | array | Same but by company name string | `["Talentify", "Workable"]` |
| `location` | array | Geo of the person | `["France", "United Kingdom", "Paris"]` |
| `industry` | array | LinkedIn industry of the person | `["Software Development", "Human Resources"]` |
| `network` | string | Connection degree — single value | `"F"` (1st), `"S"` (2nd), `"O"` (3rd+). Use `"S"` for best accept rate |
| `keyword` | array | Free-text full-profile search | Only when role isn't well-defined by `title` |
| `school_url` | array | Alumni filter — people who studied at X | `["https://www.linkedin.com/school/hec-paris/"]` |
| `past_company` | array | Ex-employees of a company (e.g. competitor) | `["https://www.linkedin.com/company/competitor/"]` |
| `connection_of` | array | People connected to a specific profile | `["https://www.linkedin.com/in/mutual-friend/"]` |
| `follower_of` | array | Followers of a thought leader / brand | `["https://www.linkedin.com/in/influencer/"]` |
| `first_name` | array | Look-alike search | `["Othamar", "Sameer"]` |
| `last_name` | array | Look-alike search | `["Magiatis"]` |

For a standard outreach campaign, the high-precision combo is:

```json
{
  "account_id": "...",
  "action": "search_people",
  "params": {
    "company_url": ["<company linkedin url>"],
    "title": ["CEO", "CTO", "Co-Founder", "Founder"],
    "network": "S",
    "limit": 5
  }
}
```

`network: "S"` (2nd degree) is gold — these people share at least one mutual connection with the user, which dramatically boosts accept rate (LinkedIn shows the mutual on the invite). Drop `network` only if the user wants 3rd-degree volume.

### Power-user patterns

- **Recruit ex-Acme talent**: `{"past_company": ["https://linkedin.com/company/acme"], "title": ["Engineer", "Senior Engineer"]}`
- **Alumni outreach to HEC grads in product roles**: `{"school_url": ["https://linkedin.com/school/hec-paris/"], "title": ["Product Manager", "CPO"]}`
- **Warm intro via mutual**: `{"connection_of": ["https://linkedin.com/in/your-investor"]}`
- **Audience of a thought leader**: `{"follower_of": ["https://linkedin.com/in/famous-vc"]}`

### Operational notes

- Cost: ~1 credit per call regardless of result count (1 credit per 10 returned, ceiling).
- Run calls **sequentially in a bash loop** (not parallel) with 1–2 sec sleeps. LinkedIn rate-limits bursts.
- Profiles shown as `LinkedIn Member` are private/anonymous → discard. Their URL contains `/search/results/people/headless?...` not `/in/<handle>`.

## Stage 4 — Enrich profiles & filter false positives (CRITICAL)

LinkedIn `search_people` regularly returns **ex-employees** of a company because they appear on the company page from past employment. You MUST verify the *current* role.

Tool: `linkedin_profiles` action `get` with `params.identifier=<linkedin handle>`.

```json
{
  "account_id": "...",
  "action": "get",
  "params": {"identifier": "<handle from profile_url>"}
}
```

- Cost: 1 credit per profile.
- For each enriched profile, parse `experience[0]` (the current job). **Keep only if** `experience[0].company` matches (or is similar to) the target company AND `experience[0].title` matches the persona titles.
- Common false positives to drop:
  - Person now at a different company (LinkedIn tagged them due to past employment).
  - Person with title "Software Engineer" / "Solutions Architect" when targeting CEO/CTO.
  - Generic "Founder @ Stealth" entries (not the target boîte).
- Surface to the user a clean two-column table: ✅ confirmed targets vs ❌ false positives with the real current job.

## Stage 5 — (Optional) Find professional emails

If the user wants email-based outreach in parallel to LinkedIn, use `linkupapi_enrich`:

```json
{"action": "find_email", "params": {"linkedin_url": "<profile_url>"}}
```

or

```json
{"action": "find_email", "params": {
  "first_name": "...", "last_name": "...",
  "company_domain": "..."
}}
```

- Cost: 1 credit per call.
- Skip this step if pure LinkedIn outreach.

## Stage 6 — Pre-flight check & send invites

Before sending **every** connection request, check the current relationship via `linkedin_network` action `check_invitation` to avoid:
- Re-inviting someone already connected (1st degree)
- Re-sending while a previous invitation is `PENDING`
- Wasting credits on `OUT_OF_NETWORK` cases

```json
{
  "action": "check_invitation",
  "params": {"profile_url": "<full linkedin url>"}
}
```

The response includes `invitation_state` which is one of:
- `NO_INVITATION` → safe to invite
- `PENDING` → already sent, skip
- `CONNECTED` (member_distance=1) → already connected, skip
- Other values → log and skip

Then for every `NO_INVITATION` profile, send the invite:

```json
{
  "account_id": "...",
  "action": "invite",
  "params": {
    "identifier": "<handle>",
    "message": "<optional ≤300 char personal note>"
  }
}
```

- **Sleep 1–2 seconds between invites** (`sleep 1` in bash). LinkedIn flags burst patterns.
- Hard cap at **80 invites per session**, **100 per rolling week** per account.
- After the batch, run `linkedin_network` action `list_sent` with `count=20` to confirm the invites appear as "Sent today".

### Message templates (when user wants a note)

Keep notes ≤ 250 chars. Patterns that work:

- **Reference shared signal**: `Hi {first_name}, I saw {Company}'s focus on {theme} — same space we're operating in. Would love to compare notes. — {sender_first_name}`
- **Genuine compliment + soft ask**: `Hey {first_name}, your work at {Company} on {topic} caught my eye. Always glad to connect with operators in {space}.`
- **Mutual connection**: `Hi {first_name}, {mutual_name} suggested I reach out. We're tackling {problem} from the {angle} side — open to a quick exchange?`

Avoid: pitchy openers, "quick question?", "5 minutes of your time?", and any link in the note (LinkedIn flags these).

## Stage 7 — Send message after accepted invitation (webhook-triggered)

If the user doesn't already have a webhook for `accepted_invitation` on the chosen account(s), create one:

```json
{
  "tool": "linkupapi_create_webhook",
  "args": {
    "account_id": "...",
    "events": ["accepted_invitation", "message_received"]
  }
}
```

- Hosted mode (no `url`) → events are stored, fetch via `linkupapi_get_webhook_events` (poll) or via SSE on the `stream_url`.
- Custom mode (with `url`) → events POSTed to your endpoint.

For each accepted invitation event you'll receive:
```json
{
  "type": "accepted_invitation",
  "new_connections": [{"profile_url": "...", "name": "...", "job_title": "..."}]
}
```

When the user wants to send a message as soon as the invitation is accepted:

```json
{
  "tool": "linkedin_messages",
  "args": {
    "account_id": "...",
    "action": "send",
    "params": {
      "profile_url": "<from event>",
      "message_text": "<personalized message>"
    }
  }
}
```

### Message-after-accept options (asked in Step 0.3 form)

The "Message-after-accept" question in Step 0.3 has exactly three options — present them in this order:

1. **Generate a short, punchy, personalized message (Recommended)** — for each accepted invite, build a custom 2–4 sentence message using the connection's `name`, `job_title`, current `Company` (from Stage 4 enrichment), and the campaign theme. **Do NOT send a generic "thanks for connecting"** — it tanks reply rate. Aim for: a specific hook tied to their work + one sharp value-prop sentence + a low-friction question. Keep ≤ 600 chars. Generate at send-time, not in advance, so the message references real profile context. Example shape:
   > {first_name}, saw you're {role} at {Company} — we just shipped a way for teams like yours to {specific value tied to theme}. Curious if {pain_point_question}?

2. **Custom template provided by the user** — the user pastes a template with `{first_name}`, `{Company}`, `{job_title}`, `{theme}` placeholders. Substitute and send verbatim.

3. **No message** — webhook still arms (so accepts are tracked) but no DM is sent. The user handles messaging manually.

Send timing: fire the message immediately when the `accepted_invitation` event arrives (no artificial 24h wait — modern outreach plays react fast, and the value-add hook makes the message feel intentional, not robotic).

## Stage 8 — Report & persist (mandatory)

At the end of every campaign run, give the user a concise summary AND **always** persist a full log file. Don't ask the user — just write it. The log is the source of truth for funnel analysis, accept-rate tracking, and de-duplication on the next campaign.

### 8a — Concise on-screen summary

```
Campaign: {ICP description}
Account:  {account_email} ({account_id})
Companies searched:    {n}
Companies kept:        {n} ({n_dropped} filtered out)
Profiles enriched:     {n}
Decision makers:       {n_confirmed} (✅) / {n_false_pos} (❌)
Skipped (already inv): {n}
Invitations sent:      {n}
Credits consumed:      {total} (start: {b0}, end: {b1})
Webhook armed:         yes/no
```

### 8b — Persist the campaign log

Write the full campaign details to `./campaigns/{YYYY-MM-DD}-{slug}.md` where `slug` is a 2-3 word kebab-case theme (e.g. `sales-rectech`, `recruit-engineers-paris`, `vc-network-fr`). If `./campaigns/` doesn't exist, create it.

The log file MUST include:

1. **ICP block** — every Stage 0 answer (goal, theme, sectors, sizes, geo, personas, volume, note style, message-after-accept).
2. **Account(s) used** — id + email + country. If multiple accounts, list per-account assignment.
3. **Funnel table** — counts and credit cost per stage.
4. **Companies kept** (Stage 2) — names only, comma-joined.
5. **Companies dropped** (Stage 2) — name + 1-line reason each.
6. **False positives dropped pre-enrichment** (Stage 3 → 4) — name + reason.
7. **Skipped (already invited / connected)** — name + state + invitation_id if available.
8. **Invites sent table** — `# | Name | Company | Network | invitation_urn`. The `invitation_urn` is the key for later `withdraw` calls and accept-tracking.
9. **Notes / monitoring** — any clusters (multiple invites in same small company), 3rd-degree count, and what to watch for in the next 48-72h. Include the snippet for arming a webhook later if Message-after-accept was "No message".

This file is what the user reads in 7 days to compute accept rate and decide whether to scale or change the ICP.

### 8c — De-dup on next campaign

When the user runs `/linkedin-outreach` again on the same account, **before Stage 6**, glob `./campaigns/*.md` for `invitation_urn`/`profile_url`/`public_identifier` of any candidate in the new batch. If a profile was already invited in any prior log, drop it from the invite list (even if `check_invitation` returns NO_INVITATION — that means LinkedIn auto-withdrew an old invite or the prior one was rejected; re-inviting too fast tanks accept rate). Surface the dedup count in the on-screen summary.

## Common pitfalls

- **Counting credits wrong** — `search_people` and `search_companies` are charged per page (10 results = 1 credit, ceiling). `get` and `invite` are 1 credit flat. Check `_credits_consumed` in every response.
- **Inviting from `LinkedIn Member` URL** — those are anonymized, the URL contains `/search/results/people/headless?...` not `/in/<handle>`. The invite call will fail. Filter them in Stage 4.
- **Trusting search_people's listed company** — always verify current role in Stage 4. ~30–50% of `search_people` results in larger companies are ex-employees.
- **Sending invites in parallel** — LinkedIn rate-limits. Use a sequential loop with sleep.
- **Greek/Cyrillic identifiers** — when the URL is URL-encoded (`https://www.linkedin.com/in/%CE%B4...`), pass the raw handle to `identifier`, not the URL.
- **Forgetting to confirm ICP** — running a 50-credit campaign on the wrong persona is bad UX. Always echo the ICP in plain text and wait for "yes" before Stage 1.

## Tool quick reference

| What you want | Tool | Action | Cost |
|---|---|---|---|
| List my connected accounts | `linkupapi_list_accounts` | — | 0 |
| Login to a new LinkedIn account | `linkupapi_login` / `linkupapi_checkpoint` | — | 1 each |
| Check credits | `linkupapi_get_credits` | — | 0 |
| Find target companies | `linkedin_profiles` | `search_companies` | 1/10 results |
| Find people in a company | `linkedin_profiles` | `search_people` | 1/10 results |
| Get full profile info | `linkedin_profiles` | `get` | 1 |
| Find professional email | `linkupapi_enrich` | `find_email` | 1 |
| Validate an email | `linkupapi_enrich` | `validate_email` | 1 |
| Reverse email → person | `linkupapi_enrich` | `reverse_email` | 1 |
| Check invitation status | `linkedin_network` | `check_invitation` | 1 |
| Send connection request | `linkedin_network` | `invite` | 1 |
| List sent invitations | `linkedin_network` | `list_sent` | 1/50 results |
| Send a DM | `linkedin_messages` | `send` | 1 (+1 if media) |
| Create webhook | `linkupapi_create_webhook` | — | 0 |
| Poll webhook events | `linkupapi_get_webhook_events` | — | 0 |

## Default campaign template (50 prospects, no note)

```
Stage 0 → confirm ICP with user
Stage 1 → search_companies (limit=25, ~3 credits)
Stage 2 → LLM filter to ~12-18 real fits
Stage 3 → search_people on each (~12-18 credits)
Stage 4 → get on every candidate (~40-60 credits)
Stage 5 → skip (LinkedIn-only campaign)
Stage 6 → check_invitation + invite on confirmed targets (~50-70 credits)
Stage 7 → ensure webhook is armed
Stage 8 → report

Total budget: ~110-150 credits for ~30-40 sent invites.
```
