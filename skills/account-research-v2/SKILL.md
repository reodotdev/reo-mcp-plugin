---

## name: account-research-v2
description: >
  Generate a comprehensive, visual account research dashboard for any company using the fast
  reo_get_account_research composite endpoint. Use this skill whenever the user asks to "research
  an account", "pull up account intel", "get me a brief on [company]", "show me what's going on
  at [company]", "run account research", or anything similar. Also trigger this skill when the
  user mentions a company name or domain and wants sales context, buyer intel, hiring signals,
  engagement activity, or contact information. Even vague prompts like "what do we know about
  [company]?" or "prep me for my call with [company]" should trigger this skill. Output is always
  a dark-mode HTML dashboard rendered inline.
compatibility: "Requires reo-gateway MCP (reo_get_account_research) and web_search tools."

# CONFIG

# ──────────────────────────────────────────────

SIGNAL_LOOKBACK: 90 days
NEWS_LOOKBACK: 30 days
MAX_CONTACTS_PER_SECTION: 5
HIRING_LOOKBACK: 90 days

HIGH_INTENT_PAGES:

- "Website: Pricing"
- "Form: Cloud PAYG"
- "Form: Self-hosted PAYG"
- "Form: Contact Sales"
- "Website: Case studies"

INCLUDE_SECTIONS:
  header: true
  overview: true
  scores: true
  activity_summary: true
  deanonymized_contacts: true
  open_data_contacts: true
  hiring_signals: true
  news_trigger_events: true
  competitive_context: true
  buyer_kpis: true

COLD_ACCOUNT_BEHAVIOUR: show_with_badge

# ──────────────────────────────────────────────

# END CONFIG

# ──────────────────────────────────────────────

# Account Research v2

This skill generates a full-page, dark-mode HTML account intelligence dashboard. It uses the
`reo_get_account_research` composite endpoint — a single API call that returns firmographic data,
engagement scores, activity feed, developer contacts, LinkedIn prospects, and hiring signals all
at once. This replaces the old pattern of making 5-7 separate reo API calls.

The entire reo data-gathering step is ONE tool call. Claude's job is to:

1. Call `reo_get_account_research` once
2. Run 2-3 web searches for news, competitive context, and buyer KPIs
3. Write inference/analysis blocks based on the data
4. Render the HTML dashboard

## Step 1 — Resolve Domain

If the user gives a company name but not a domain, infer the most likely domain (e.g. "Vercel" →
"vercel.com"). If ambiguous, ask before proceeding.

## Step 2 — Fetch All Reo Data (single call)

Call `reo_get_account_research` with these parameters:

```
reo_get_account_research(
  domain = "<company domain>",
  signal_lookback_days = 90,
  max_contacts = 5,
  high_intent_pages = ["Website: Pricing", "Form: Cloud PAYG", "Form: Self-hosted PAYG", "Form: Contact Sales", "Website: Case studies"],
  prospect_seniority_filter = ["VP", "Director", "Head", "CXO"]
)
```

This single call returns everything you need from reo:


| Response Section | What It Contains                                                                |
| ---------------- | ------------------------------------------------------------------------------- |
| `firmographic`   | Company name, industry, HQ, tech headcount, tech stack, ICP fit, dev stage      |
| `engagement`     | Score (0-100), intent level, event counts, high-intent hits, activity breakdown |
| `top_activities` | Up to 15 recent activities with actor name, type, page, source, timestamp       |
| `developers`     | Up to 5 deanonymized developers with email, activity summary, notable pages     |
| `prospects`      | Up to 5 LinkedIn prospects with title, seniority, LinkedIn URL                  |
| `hiring`         | Total open roles, breakdown by tech function, date range                        |


If any section is empty, display "— Not available" in the dashboard rather than skipping the section.

## Step 3 — Web Search Supplement (parallel)

Run these web searches in parallel — they supplement the reo data with external context:

### 3a. Competitive Context

Search for tools/vendors the company uses in your category. Look for job descriptions, LinkedIn
posts, tech blog posts. For each competitor found: name, status (Active/Primary, Partial, Churned,
Complementary), 2-sentence description. Include objection prep if a primary competitor is found.

### 3b. News & Trigger Events

Search for news from the last 30 days: funding, exec hires, product launches, outages. For each
event: icon, title, 1-sentence sales implication, date. Omit section if nothing found.

### 3c. Buyer KPIs by Role

Use the contacts found in the reo response to identify buyer personas. For each, state their
likely KPIs based on role + industry. Brief web search if needed for industry-specific KPIs.

## Step 4 — ICP Fit Score

Compute ICP Fit (0-100) using the reo data + the `.me` file (if available):

- Read `.me` to find ICP definition (industry, team size, tech stack, signals)
- Compare against firmographic data: `tech_headcount`, `industry`, `technology_names`
- Compare against hiring data: `tech_function_counts` patterns
- Compare against engagement signals
- Score each dimension, weight, and total
- Label confidence: High (all dimensions available), Medium (some missing), Low (most missing)
- more 

The `customer_fit_score` from reo (e.g. "STRONG", "MODERATE") can be used as a sanity check,
but compute your own score for the detailed breakdown.

If no `.me` file exists, use `customer_fit_score` from reo directly and note that a custom ICP
profile would enable a more detailed breakdown.

## Step 5 — Write Inference Blocks

These are where Claude adds real value beyond raw data. Be specific and opinionated — tie
observations to actual people, actions, and dates from the data. No generic filler.

### Activity Summary — "What this suggests"

2-4 sentences on what the activity pattern means. Consider: velocity (accelerating/stalling?),
breadth (one person or many?), depth (deep in docs or just browsing?), intent signals (pricing/demo
hits?). Connect to a likely sales scenario.

### Developer Contacts — "What this tells us"

Synthesise what the collective developer behaviour signals. Examples of the kind of specificity
expected:

- Multiple engineers with COPY_TEXT on CLI pages → active eval, likely POC
- Senior engineer + procurement visitor on pricing → evaluation committee forming
- Activity spread over 3+ months with no high-intent signals → slow eval, needs nudge

### Suggested Next Steps

3-5 concrete, prioritised actions. Each includes:

- **Action**: specific to actual people and signals (e.g. "Send CLI quickstart to Tyler Schultz — 
copied 5 commands last week")
- **Why**: 1-sentence rationale tied to a signal
- **Urgency**: 🔴 This week / 🟡 This month / ⚪ When ready

Order by urgency. Highest-signal, lowest-effort first. If account is cold, suggest prospecting
actions based on hiring signals and contacts.

## Step 6 — Render the HTML Dashboard

Generate a full dark-mode HTML dashboard. Self-contained (all CSS inline, no external deps).

### Design Specifications

**Color palette:**

- Background: `#1a1a1a` (page), `#242424` (card), `#2d2d2d` (nested card)
- Text primary: `#e8e8e8`; secondary: `#9a9a9a`; label: `#6b6b6b` (uppercase, 11px, 0.08em tracking)
- Accent green: `#4ade80`; blue: `#60a5fa`; orange: `#f59e0b`; red: `#f87171`
- Score bar track: `#3a3a3a`; ICP fill: `#4ade80`; Engagement fill: `#60a5fa`
- Competitor status: Active=`#f87171`, Partial=`#f59e0b`, Churned=`#4ade80`, Complementary=`#9a9a9a`
- Hiring badge: dark green bg `#14532d`, green text `#4ade80`
- High-intent banner: `#14532d` bg, `#4ade80` text
- Objection banner: `#422006` bg, `#f59e0b` text
- Activity dots: blue `#60a5fa` (product), teal `#2dd4bf` (email), orange `#f59e0b` (event), grey `#6b6b6b` (outbound)

**Typography:** system-ui / sans-serif. Section labels: 11px uppercase 0.08em tracking. Body: 13-14px.

**Layout:** Single-column scroll. Cards with `border-radius: 12px`. Max-width 820px centered.

### Dashboard Sections (in order)

1. **Header** — Company initials avatar, company name, tagline (industry · HQ · engagement score),
  status tags (ICP level, Intent level), last updated timestamp.
2. **Overview** — Row 1: Industry, Tech headcount, Founded Year, HQ. Row 2: Tech stack badges.
  Missing fields show "— Not available" in muted grey.
3. **Scores** — Two-panel row:
  - Left: ICP Fit score bar + breakdown of matched/unmatched criteria
  - Right: Engagement score bar, Intent level badge, Dev Stage from reo
4. **Activity Summary** — Chronological feed from `top_activities`: colored dot, actor name,
  activity type, page label, source, date. Below: **"What this suggests"** inference block.
   Cold badge if empty.
5. **Developer Champions** — From `developers`: avatar initials, name, email, title, activity
  summary (counts by type), notable pages, first/last seen. Below all cards: **"What this tells
   us"** inference block. Cold badge if none.
6. **More Prospects** — From `prospects`: avatar initials, name, title, location, LinkedIn URL
  (linked), outreach note.
7. **Hiring Signals** — Total open roles, tech function badges with counts, insight banner.
8. **News & Trigger Events** — From web search. Icon, title, implication, date. Omit if empty.
9. **Competitive Context** — From web search. Competitor cards with status. Objection prep if
  applicable.
10. **Buyer KPIs by Role** — 2×2 grid: role label, KPI list.
11. **Suggested Next Steps** — 3-5 action cards with urgency badges.

## Error Handling

- Empty reo section → show "— Not available" in muted grey, never skip a section
- Thin web search results → show what's available, note it's limited
- Never hallucinate contacts, scores, or activities — only display what was returned
- Unresolvable domain → ask user to confirm before proceeding

## Technical Notes

**Response field reference** — The `reo_get_account_research` response uses these exact field paths:

- `meta.org_id` — UUID for the account
- `firmographic.company_name`, `.industry`, `.hq.city/.state/.country`
- `firmographic.tech_headcount` — integer, use for "Tech Headcount" (not estimated_employee_range)
- `firmographic.technology_names` — array of strings, render as badges
- `firmographic.customer_fit_score` — string like "STRONG", "MODERATE"
- `firmographic.developer_stage` — string like "HIGH", "LOW"
- `firmographic.developer_activity_score` — integer 0-100
- `engagement.score` — pre-computed 0-100
- `engagement.intent_level` — "High", "Medium", "Low"
- `engagement.high_intent_hits[]` — array of {page, date, actor}
- `engagement.activity_type_breakdown` — object with type→count
- `top_activities[]` — {actor_name, activity_type, page, activity_source, timestamp}
- `developers[]` — {name, email, title, linkedin, github, location, first_seen, last_seen, signal_strength, activity_score, activity_summary, notable_pages, reo_link}
- `prospects[]` — {name, title, seniority, tech_function[], location, linkedin, email}
- `hiring.total_open_roles`, `.by_tech_function[].function/.count`, `.date_range`

**Activity signal hierarchy** (for prioritising and narrating):
FORM_CAPTURE > DEMO_LOGIN > COPY_TEXT/COPY_COMMAND > PAGE_VISIT > Identity

**COPY_TEXT and COPY_COMMAND** both indicate someone copied something — these are implementation
signals stronger than PAGE_VISIT. Treat them as active evaluation indicators.