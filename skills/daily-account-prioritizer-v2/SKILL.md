---

## name: daily-account-prioritizer-v2
description: >
  Generates a holistic, scored, SDR/AE-ready list of accounts to focus on today —
  combining ICP fit, dev funnel stage, activity score, and recency momentum signals
  from reo, with territory and CRM stage acting as hard filters. Uses the fast
  reo_get_daily_focus_list composite endpoint — a single API call that returns
  pre-scored, pre-ranked, pre-enriched accounts. Use this skill whenever someone
  asks: "what accounts should I focus on today?", "give me my daily account list",
  "who should I call today?", "prioritize my accounts", "show me my hot list",
  "rank accounts by fit", "build my SDR list", "score my accounts", or any request
  for a ranked or scored account list. Always trigger this skill for daily pipeline
  triage workflows.
compatibility: "Requires reo-gateway MCP (reo_get_daily_focus_list)."

# CONFIG

# ──────────────────────────────────────────────

TERRITORY_DEFAULT: "ask user"
TOP_N: 20
RECENCY_DAYS: 7

EXCLUDED_CRM_STAGES:

- Customer
- Closed Lost
- Churned
- Disqualified
PROSPECT_SENIORITY:
- Founder
- C-Level
- VP
- Director
- Head

# ──────────────────────────────────────────────

# END CONFIG

# ──────────────────────────────────────────────

# Daily Account Prioritizer v2

This skill produces a ranked, scored list of accounts for an SDR or AE to act on
today. It uses the `reo_get_daily_focus_list` composite endpoint — a single API
call that handles all data fetching, filtering, scoring, and enrichment server-side.

Claude's job is:

1. Confirm the user's territory
2. Call `reo_get_daily_focus_list` once
3. Write "Why it's here" and "Recommended action" for each account
4. Render the HTML dashboard
5. Provide a narrative summary

## Scoring Overview (100 points total)

The composite endpoint computes all scores server-side. Claude does NOT need to
calculate any of these — they arrive pre-computed in the response. This reference
is for understanding the dashboard display:


| Component      | Weight | What it measures                                                    |
| -------------- | ------ | ------------------------------------------------------------------- |
| ICP fit        | 20     | Customer fit tier (STRONG=20, MODERATE=10)                          |
| Funnel fit     | 25     | Developer stage (DEPLOY=25, BUILD=17, etc.)                         |
| Activity score | 25     | Developer activity score × 0.25, capped at 25                       |
| Recency score  | 30     | Activity delta (15) + Mgr+ identified (10) + Δ deanon devs (5, n/a) |


Max attainable: 95 (Δ deanon dev scoring is unavailable).

## Step 0 — Confirm Territory

Ask once: *"What's your territory? (e.g. US + Canada, EMEA, or 'global')"*

Wait for the answer unless it's obvious from context or a prior conversation.
Map the answer to a `country_list` array:

- "US + Canada" → `["United States", "Canada"]`
- "EMEA" → `["United Kingdom", "Germany", "France", "Netherlands", ...]`
- "global" → omit the parameter entirely

## Step 1 — Fetch the Focus List (single call)

Call `reo_get_daily_focus_list` with:

```
reo_get_daily_focus_list(
  country_list = ["United States", "Canada"],   // from Step 0
  top_n = 20,
  recency_days = 7,
  customer_fit_score = ["STRONG", "MODERATE"],
  excluded_crm_stages = ["Customer", "Closed Lost", "Churned", "Disqualified", "CUSTOMER"]
)
```

This single call returns everything:


| Response Field                    | What It Contains                                                                                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `meta`                            | Timestamp, territory, filters applied, pool stats                                                                                                         |
| `meta.pool_stats`*Top activities | total_accounts_fetched, after_hard_filters, stage_a_shortlist, final_top_n                                                                                |
| `accounts[].scores`               | Pre-computed: icp_fit, funnel_fit, activity_score, stage_a, delta_activity, delta_activity_pts, mgr_plus, mgr_plus_name, mgr_plus_pts, recency, composite |
| `accounts[].top_activities`       | Up to 3 recent high-signal activities per account                                                                                                         |
| `accounts[].developers`           | Up to 3 deanonymized developers with email, title, LinkedIn                                                                                               |
| `accounts[].prospects`            | Up to 3 LinkedIn prospects with tech_function_match flag                                                                                                  |


If the response has fewer than 5 accounts, note it in the dashboard banner. The
endpoint has already applied all filters — a small result set means the territory
or recency window is genuinely narrow.

## Step 2 — Write Account Intelligence

For each account, Claude adds two pieces of natural language intelligence that
the endpoint cannot generate:

### "Why it's here" — one concrete sentence per account

This must be specific to actual signals, people, and dates. It is the single most
important line on each card — an SDR will decide whether to act based on this.

Good examples:

- "Wes Anderson copied from the documentation page this week while the CEO's activity score hit 100 — active evaluation."
- "CMO Gaurav Deshpande has a perfect activity score and Shaneeza Mohamed is building automations — multiple teams engaged."
- "Founder Ting Wang is personally active on the platform with 100 activity score — founder-led evaluation."
Bad examples (never produce these):
- "This account has high activity." ← too generic
- "Strong ICP fit and good engagement." ← says nothing actionable

### "Recommended action" — one prescriptive line per account

Base this on the combination of funnel stage, scores, and people identified:


| Signal Pattern                            | Action Template                              |
| ----------------------------------------- | -------------------------------------------- |
| DEPLOY + high recency                     | "Call today — expansion signal"              |
| DEPLOY/BUILD + Mgr+ identified            | "Book a meeting with [mgr_plus_name]"        |
| High activity + multiple developers       | "Multi-thread: reach [dev1] and [prospect1]" |
| DISCOVER + strong ICP + rising activity   | "Send targeted content + monitor this week"  |
| Strong ICP + flat activity                | "Warm outbound to [prospect name]"           |
| High activity + MODERATE ICP              | "Qualify before investing time"              |
| prospects with tech_function_match = true | Prioritize those contacts in the action      |


## Step 3 — Render the HTML Dashboard

Generate a single self-contained dark-mode HTML dashboard file. Save to outputs
folder and share the link.

### Header

- Title: *"🎯 Today's Focus List — [Date]"*
- Subheader: territory · eligible pool size · shortlist size · data freshness
- Legend: ICP 20 · Funnel 25 · Activity 25 · Recency 30
- Banner: *"Δ deanon dev scoring (3E) unavailable — max attainable score: 95"*
(from `meta.scoring_notes`)

### One Card Per Account (ranked 1–N)

Each card contains:

1. **Header row** — rank, company name, domain, composite score with a horizontal bar.
  Bar colours: green ≥ 70 · amber 45–69 · red < 45.
2. **Score breakdown** — four pills:
  *"ICP 20/20 · Funnel 25/25 · Activity 18/25 · Recency 22/30"*
   Compact recency line: *"Δ activity +15 · Mgr+ ✓ [name]"* or *"Mgr+ ✗"*
3. **"Why it's here"** — the sentence from Step 2.
4. **Top activities** — from `accounts[].top_activities`. Each row:
  Fetch the accounts activity summary of last 7 days  
   If empty (upstream API unavailable): show *"Activity detail temporarily unavailable"*
5. **Known developers** — from `accounts[].developers`. Each row:
  `name · title · signal_strength · LinkedIn ↗`
   If title is empty, show email instead. If no developers: *"— not yet identified"*
6. **More prospects** — from `accounts[].prospects`. Each row:
  `name · title · seniority · LinkedIn ↗`
   If `tech_function_match` is true, show a small badge: *"✓ tech match"*
   If no prospects: *"— none found at this seniority level"*
7. **Recommended action** — the prescriptive line from Step 2.

### Styling

- Background `#0f0f12` · card `#1a1a22` · border `#27272e`
- Accent `#6366f1` · text `#e5e7eb` · muted `#9ca3af`
- Score bar: green `#22c55e` · amber `#f59e0b` · red `#ef4444`
- Tech match badge: `#14532d` bg, `#4ade80` text
- Readable sans-serif, generous whitespace. No emoji soup.
- Max-width 900px centered. Cards with `border-radius: 12px`.

## Step 4 — Narrative Summary

3–5 sentences in chat alongside the dashboard link:

- Dominant pattern across the top accounts (e.g., "heavy on DEPLOY-stage accounts")
- Single most urgent action and why
- Notable outlier (e.g., a cold account that suddenly appeared, a founder
personally evaluating)
- Quick note on pool health: how many survived filters vs total

## Edge Cases

- **Fewer than 5 accounts returned**: show a banner noting the pool is thin.
Suggest the user try widening territory or relaxing the recency window.
- `**top_activities` empty for some accounts**: the upstream activities API may
be temporarily unavailable for some accounts. Show *"Activity detail temporarily
unavailable"* instead of leaving the section blank.
- **All `tech_function_match` = false**: common for small companies where
leadership is tagged "Not sure". Don't flag this as a problem — just omit the
badge and show prospects normally.
- `**org_name` blank**: show domain only, labelled "(unnamed)".
- `**mgr_plus_name` references "QA Lead" or similar non-manager title**: the
endpoint's keyword matching is broad. Don't second-guess it in the dashboard —
show the name as-is.

## Technical Notes

**Response field reference** — exact paths in the `reo_get_daily_focus_list` response:

- `meta.generated_at` — ISO timestamp
- `meta.territory` — array of countries or ["global"]
- `meta.pool_stats.total_accounts_fetched` — raw pool size
- `meta.pool_stats.after_hard_filters` — after CRM + recency filters
- `meta.pool_stats.final_top_n` — number of accounts in response
- `meta.scoring_notes[]` — array of strings, display as banners
Per account:
- `rank`, `org_id`, `domain`, `org_name`, `industry`, `location`
- `estimated_employee_range`, `developer_stage`, `customer_fit`
- `deanonymised_dev_count_range`
- `scores.icp_fit` (max 20), `scores.funnel_fit` (max 25)
- `scores.activity_score` (max 25), `scores.stage_a` (max 70)
- `scores.delta_activity` (raw delta), `scores.delta_activity_pts` (max 15)
- `scores.mgr_plus` (bool), `scores.mgr_plus_name` (string)
- `scores.mgr_plus_pts` (0 or 10), `scores.recency` (max 25 effective)
- `scores.composite` (max 95)
- `top_activities[]` — {actor_name, activity_type, page, activity_source, timestamp}
- `developers[]` — {name, email, title, linkedin, signal_strength, activity_score}
- `prospects[]` — {name, title, seniority, tech_function, location, linkedin, email, tech_function_match}
**Activity signal hierarchy** (for narrating "Why it's here"):
FORM_CAPTURE > DEMO_LOGIN > COPY_TEXT/COPY_COMMAND > PAGE_VISIT > Identity

**Relative time display**: convert `timestamp` to relative format for the dashboard:

- Today → "today", yesterday → "1d ago", etc. Use activity_source date, not render time.

