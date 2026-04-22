---
name: sales-call-prep
description: >
  Preps customer meetings for a sales rep — fetches events from Google Calendar
  (today, a specific day, or the whole week) and generates a structured brief
  for each meeting using the reo-gateway MCP. Each brief combines per-meeting
  account intel (engagement, signals, hiring, prospects) with per-attendee
  intel matched by email and name. Supersedes the older call-prep-sheet and
  weekly-meeting-digest skills that were built on the legacy reo MCP.
  Trigger this skill whenever the user asks to "prep my calls", "prep today",
  "prep my week", "what's on my calendar", "get me ready for my meetings",
  "prep [company] call", "Monday briefing", "call prep", or any phrasing that
  implies "help me get ready for the customer meetings on my calendar". Also
  trigger when the user references a specific meeting they have coming up and
  wants background on it.
compatibility: Requires Google Calendar MCP and reo-gateway MCP. Optionally uses web_search for news.
---

# ═══════════════════════════════════════════
# CONFIG — edit the values here to customise the skill
# ═══════════════════════════════════════════
#
# HOW TO UPDATE THIS CONFIG
# ─────────────────────────
# 1. Edit only the value to the right of the colon.
# 2. Do not rename parameters — the skill reads them by name.
# 3. Save the file. Changes apply on the next run.
#
# ADDING YOUR COMPANY'S DOMAIN
# ─────────────────────────
# Add your company's email domain(s) to INTERNAL_DOMAINS so internal-only
# meetings are skipped. Any meeting where every attendee is on an internal
# domain is dropped from the brief.
# ═══════════════════════════════════════════

SCOPE_DEFAULT: ask
# What to do when the user doesn't specify a time range.
#   "ask"        → ask the user (today vs whole week) before fetching the calendar
#   "today"      → default to today's meetings
#   "tomorrow"   → default to tomorrow's meetings
#   "week"       → default to the current work week (Mon–Fri)
# The user can always override inline: "prep tomorrow", "prep Friday", "prep this week".

WEEK_START_DAY: Monday
# Which day the work week starts on. "Monday" (default) or "Sunday".
# Only used when scope = week.

SIGNAL_LOOKBACK_DAILY: 30 days
# How far back to pull account activity signals when the scope is a single day.
# Longer window = better context for slow-moving evaluations.

SIGNAL_LOOKBACK_WEEKLY: 7 days
# How far back to pull account activity signals when the scope is a whole week.
# Shorter window = "what happened this week" framing.

MAX_CONTACTS: 5
# How many developers/prospects to pull from reo_get_account_research per company.
# Keep this tight — we match calendar attendees against this list first before
# falling back to the broader reo_list_prospects search.

PROSPECT_SEARCH_PAGE_SIZE: 100
# If an attendee isn't matched against the top-N developers/prospects from
# reo_get_account_research, the skill falls back to reo_list_prospects with this
# page size and does a fuzzy name match. Higher = more likely to find junior
# attendees, but slower.

INTERNAL_DOMAINS:
  - "reo.dev"
# Email domains belonging to your company. Meetings where ALL attendees are
# on these domains are skipped. Add every domain your company uses.

FREE_EMAIL_DOMAINS:
  - "gmail.com"
  - "googlemail.com"
  - "outlook.com"
  - "hotmail.com"
  - "yahoo.com"
  - "icloud.com"
  - "me.com"
  - "aol.com"
  - "proton.me"
  - "protonmail.com"
# External attendees on these domains don't map to a company — the skill
# flags them as "personal email — company unknown" rather than trying to
# run account research on gmail.com.

HIGH_INTENT_PAGES:
  - "Website: Pricing"
  - "Form: Cloud PAYG"
  - "Form: Self-hosted PAYG"
  - "Form: Contact Sales"
  - "Website: Case studies"
# Passed to reo_get_account_research as high_intent_pages. Used to flag the
# single highest-intent activity in each brief — the "don't miss" moment.

INCLUDE_SECTIONS:
  attendee_intel: true       # Per-person lookup: reo activity + LinkedIn profile
  company_overview: true     # Industry, headcount, tech stack from reo
  engagement_score: true     # reo engagement score + intent level
  signals: true              # Account-level activity signals
  highest_intent: true       # Spotlight the single most commercially important signal
  hiring: true               # Open hiring roles and what they imply
  news: true                 # Recent news from web search (if web_search available)
  suggested_question: true   # Generate an opening question tailored to the signals
  watch_out_for: true        # Flag red flags (cooling signals, key person absent, etc.)
# Set any section to false to remove it from all briefs. Useful if you want a
# faster, lighter prep (e.g. turn news off to skip web search entirely).

DEEP_ENRICH: false
# When true, after matching an attendee we also call reo_get_enriched_profiles
# for a richer LinkedIn profile snapshot. Off by default — it adds calls for
# marginal value. Turn on for high-stakes meetings.

COLD_ACCOUNT_BEHAVIOUR: show_news_only
# What to do when a company has zero reo signals.
#   "show_news_only" → skip signals section, still show news + hiring
#   "skip"           → omit the company entirely from the brief
#   "flag"           → show the brief but mark it clearly "cold — no product signals"

NEWS_PER_COMPANY: 2
# Number of web_search headlines to surface per company.
# 0 = disable web search entirely (faster, no news section).

WATCH_LIST_SIZE: 5
# In week mode, how many "hot but no meeting" accounts to list at the bottom.
# Accounts with strong recent signals that aren't on your calendar this week.
# 0 = disable the watch list.
#
# Note: populating the watch list requires scanning many accounts, which today
# means the skill skips it unless reo_get_daily_focus_list or an equivalent
# bulk endpoint is available. Treat this as a soft cap.

# ═══════════════════════════════════════════
# END CONFIG
# ═══════════════════════════════════════════


# Sales Call Prep — Skill Instructions

## What this skill does

Turns a calendar view into a stack of sales-ready meeting briefs. For each
external customer meeting it surfaces:

- **Attendee intel** — matched against reo's deanonymized developers by email,
  then against prospects by name. Title, seniority, LinkedIn, product activity.
- **Account context** — firmographic data, engagement score, recent activity
  signals, open hiring roles.
- **Don't-miss moments** — the single highest-intent thing happening on the
  account right now (e.g. "3 engineers on the pricing page yesterday").
- **News** — 1–2 recent headlines with sales implications.
- **Suggested opening + red flags** — one specific question to open the call;
  any warnings worth knowing before you dial in.

Scope is flexible: today, tomorrow, a specific day, or the whole week.

## Step 1 — Determine the scope

Parse the user's request for a time range.

- Explicit day ("today", "tomorrow", "Friday", "April 10") → use that.
- Explicit range ("this week", "next week", "Mon–Fri") → use that.
- Nothing specified → apply `SCOPE_DEFAULT`.
  - `ask` → ask the user: "Should I prep today's calls, or the whole week?"
  - Otherwise use the configured default.

If the request is on a weekend and scope = week, default to the *upcoming*
week (Mon–Fri). Make that assumption visible in the output header so the user
can correct you.

## Step 2 — Fetch calendar events (Google Calendar MCP)

Fetch events from the primary calendar for the target date range.

Drop events that aren't customer-facing:
- All attendees are on `INTERNAL_DOMAINS` → internal meeting.
- Event is declined, "busy" placeholder, or OOO.
- No external attendees at all.

A customer-facing event usually has a title like *call, demo, sync, intro,
discovery, review, walkthrough, check-in, catch-up, QBR, kickoff, onboarding*.
Use this as a soft filter — if a meeting has external attendees but an unusual
title, include it anyway; it's better to include one odd internal-looking
event than to drop a real call.

For each qualifying event extract:
- Start time, duration, title, whether it's recurring
- Attendee list: `[(display_name, email, response_status), ...]`, external only
- Set of external company domains (derived from email domains, excluding
  `INTERNAL_DOMAINS` and `FREE_EMAIL_DOMAINS`)

If a meeting has zero usable external domains (e.g. all attendees are on
gmail), emit a brief that only lists the attendees with a "personal email —
company unknown" note. Don't run account research on free-email domains.

If no qualifying events exist in the range: say so plainly ("No customer calls
on your calendar for [range]") and stop. Don't fabricate a brief.

## Step 3 — Account research per company (reo_get_account_research)

For each unique external company domain across all meetings, make ONE call:

```
reo_get_account_research(
  domain = "<company domain>",
  signal_lookback_days = <SIGNAL_LOOKBACK_DAILY or SIGNAL_LOOKBACK_WEEKLY based on scope>,
  max_contacts = <MAX_CONTACTS>,
  high_intent_pages = <HIGH_INTENT_PAGES>
)
```

This single call returns firmographic, engagement, top activities, developers
(with email), prospects (with LinkedIn public_id), and hiring — everything the
brief needs about the account.

Run these calls in parallel across companies when possible. Cache results in
memory so a company that appears on two days of the week isn't queried twice.

If the call fails for a domain (bad domain, no reo account), fall back
gracefully: show the meeting with attendee names only and a "account not found
in reo" note. Don't crash the whole brief.

Apply `COLD_ACCOUNT_BEHAVIOUR` if `engagement.total_events_in_window` is zero.

## Step 4 — Attendee matching cascade

For each external attendee in each meeting, run this cascade in order. Stop
at the first hit:

### 4a. Email match against `developers[]`

The `developers[]` array from Step 3 contains up to `MAX_CONTACTS` deanonymized
developers for the account, each with an email field. Lowercase both sides and
compare. On match, you already have:

- name, email, title (often blank), linkedin URL, github, location
- first_seen, last_seen, signal_strength, activity_score
- notable_pages, activity_summary
- A `reo_link` containing a developer UUID like `?devId=<uuid>`

If the attendee matches a developer, you have their product activity. No
follow-up call is needed for the brief. (Optional: if `DEEP_ENRICH` is true or
the user explicitly asks for deeper per-person activity, parse the UUID from
`reo_link` and call `reo_get_developer_activities(developer_id)` for the full
page-level breakdown.)

### 4b. Name match against `prospects[]` (from account_research)

The `prospects[]` array contains senior LinkedIn prospects for the account.
Their emails are null — match on `full_name` vs the attendee's display name.
Use a tolerant match: case-insensitive, ignore middle initials, ignore
suffixes (Jr., III), ignore honorifics. If name isn't available in the
calendar event, skip this step for that attendee.

On match, you have: title (`designation`), seniority, tech_function, location,
LinkedIn URL, LinkedIn public_id (the `id` field).

### 4c. Broader name search (reo_list_prospects)

If 4a and 4b both missed, call `reo_list_prospects(domain, page_size=<PROSPECT_SEARCH_PAGE_SIZE>)`
and fuzzy-match by name against the returned list. This catches junior
attendees or non-technical buyers who aren't in the senior-only default from
Step 3.

Do this lookup ONCE per company (not per attendee) — fetch the page, then
match all unmatched attendees against the result in memory.

If still no match after scanning the first page, stop. Going deeper isn't
worth the latency for call prep.

### 4d. Fallback

No match anywhere. This is common and expected — many buyers (execs,
procurement, marketing) aren't tracked by reo. Show the attendee's name + email
with a "— No reo profile or product activity on record" line. Don't apologise
for it; it's normal.

### 4e. Optional deep enrichment

If `DEEP_ENRICH` is true AND we have a LinkedIn public_id for the attendee
(from steps 4b or 4c), call:

```
reo_get_enriched_profiles(
  profiles = [{ company_domain: "<domain>", public_ids: ["<public_id>"] }]
)
```

Batch these into one call across all attendees at the same company where
possible — the `profiles` parameter takes a list.

## Step 5 — News (optional, web_search)

If `NEWS_PER_COMPANY > 0` and web_search is available: one search per company,
e.g. `"<Company Name> news"`. Extract top N headlines from the last ~30 days.
For each, write a one-sentence sales implication ("They raised a Series B →
budget may be loosening; ask about new initiatives"). Skip if search returns
nothing — don't force it.

If web_search isn't available in this environment, silently skip the news
section. Don't emit an error.

## Step 6 — Generate the brief

Format one card per meeting, ordered chronologically. If scope is a week,
group cards under day headers (Monday, Tuesday, etc.).

### Single-meeting card template

```
---
### [HH:MM] — [Company Name] ([domain]) [(recurring) if applicable]
**Meeting:** [Event title] · [Duration] · [High-intent / Active / Cold badge]

#### 👤 Attendees
**[Name]** — [Title, from reo or "role unknown"] · [City if available]
- Product activity: [summary — or "no activity on record"]
- Last active: [relative date, or "—"]
- Notable: [flag HIGH_INTENT_PAGE hits here, or omit the line]

*(Repeat per external attendee. If the attendee has only their name + email
with no reo data, show that honestly — it's useful information in itself.)*

#### 🏢 Account Overview
- Industry: [from firmographic] · Tech headcount: [N] · HQ: [city, country] · Founded: [year or "—"]
- Tech stack: [top 6-8 technology_names as a readable list, or "—"]
- ICP fit: [customer_fit_score] · Dev stage: [developer_stage]

#### 📊 Engagement *(if engagement_score enabled and signals exist)*
- Score: [0-100] · Intent: [High/Medium/Low] · Events (last [window]): [N] · Unique actors: [N]
- Last active: [date]

#### 🔥 Signals *(if signals enabled and activity exists)*
- [Page label]: [N] visits · [unique actors]
- [Activity type]: [N] · [relative date]
- Total: [N] events across [N] users in the last [window]

**⚡ Don't miss** *(if highest_intent enabled and a high_intent_hit exists)*
> [The most commercially significant thing — e.g. "2 engineers hit the pricing
> page in the last 5 days. Ask whether they're starting a formal evaluation."]

#### 💼 Hiring *(if hiring enabled)*
- [N] open roles · Top functions: [top 3 tech functions with counts]
- What this suggests: [1-sentence read, e.g. "Hiring hard on Security & Compliance
  — likely scaling prod infra. Our compliance story matters here."]

#### 📰 News *(if news enabled and results found)*
- [Headline] → [Sales implication]

#### ❓ Suggested opening question *(if suggested_question enabled)*
> "[One specific question tied to their signals, attendee activity, or news.
> Not a generic 'how's your week'.]"

#### ⚠️ Watch out for *(if watch_out_for enabled and red flags exist)*
- [e.g. "[Name] on the pricing page last week is NOT on this call — ask who else is involved"]
- [e.g. "Signal activity dropped from 14 events last month to 2 this week — momentum cooling"]

#### 🧠 Read
2–3 sentences synthesising attendees + signals + news. Reference specific
people and actions from the data — never generic ("this account is promising").
Example: "Sarah has been deep in the SSO docs and copied 3 CLI commands —
she's the technical evaluator. James isn't on this call but hit pricing twice,
which suggests he's managing budget separately. Use this call to ask Sarah
who else is on the evaluation committee."
---
```

### Week-mode layout

When the scope is a whole week:

```
# Week of [Mon date] – [Fri date]
[N] customer calls · [N] unique accounts · scope: [this week | next week | explicit range]

---

## MONDAY, [Date]
[card]
[card]

## TUESDAY, [Date]
No customer calls scheduled.

[... etc for each day]

---

## 👀 Watch list *(if WATCH_LIST_SIZE > 0 and we have the data)*
Accounts with strong recent activity but no meeting on the calendar:

| Company | Signals ([window]) | Highest intent | Suggested action |
|---------|-------------------|----------------|------------------|

(If a bulk-account endpoint isn't available in this environment, omit the
watch list table and note "— watch list skipped; requires focus-list endpoint".)

---

## 🧭 Week summary
- **Top priority:** [Company] on [Day] at [Time] — [reason]
- **Pattern:** [2-3 sentences reading the whole-week shape — e.g. "3 of 5 calls
  hit pricing this month; momentum is real but concentrated. The watch list
  suggests two more accounts worth chasing before Friday."]
- **Before each call:** one concrete action per meeting, ordered by time.
```

## Step 7 — Output hygiene

- Use the relative-date format "2d ago", "last Wed", "3 weeks ago" — not raw
  ISO timestamps. Users are reading this 5 minutes before a call; brevity wins.
- If a section has no data, omit it rather than printing "— Not available"
  everywhere. A lean card is more useful than a sparse one.
- Never invent attendees, activity, or signals. If reo says zero, say zero.
- If a company has one attendee and they matched zero data in any of the
  cascade steps, it's still a valid card — just a short one. Say so.

## Edge cases

- **All-hands or internal-sounding titles with external attendees** (e.g. "Team
  offsite" but three external emails): include it. Better to prep for a
  surprise customer call than to drop it.
- **Recurring meeting where no new signals have appeared since last week**:
  still generate the card, but add a "- Context established; no new signals"
  line instead of a full signals block.
- **Company domain resolves to a parent** (e.g. attendee is `@aws.amazon.com`
  but reo only tracks `amazon.com`): try both — use the subdomain first, fall
  back to the registrable domain.
- **One meeting, multiple external companies** (rare but happens): generate
  one card per company, under the same meeting header. Deduplicate attendees
  across the cards (they belong to only one company).
- **reo_get_account_research succeeds but returns empty everything**: the
  domain resolved but the account has no data. Treat as cold per
  `COLD_ACCOUNT_BEHAVIOUR`.
- **Google Calendar MCP not connected**: tell the user plainly that Calendar
  access is required and stop. Don't try to proceed.
- **reo-gateway MCP not connected**: tell the user plainly and stop. The
  skill's value is the reo data — without it, there's no brief to write.

## Why this skill is structured this way

Three things are worth calling out for anyone reading or editing this skill:

1. **One account_research call per company is enough.** The gateway endpoint
   is a composite — it fans out to firmographic, developers, prospects,
   hiring, activities in parallel and returns them unified. Don't try to "save
   calls" by skipping it; don't try to call the individual endpoints
   separately either. One call, everything.

2. **The matching cascade exists because reo has two identity systems.**
   Developers are tracked by email (they used your product with identity
   resolution). Prospects are tracked by LinkedIn public_id (they're enriched
   from LinkedIn, never logged in to anything). Calendar gives us email + name.
   We try email first (strongest), then name (weaker, but works for prospects),
   then broader name search. Don't shortcut this — email-only matching will
   miss every non-developer attendee, which is the majority on discovery calls.

3. **The "unmatched attendee" case is the common case, not a failure.** Most
   meetings have at least one attendee who isn't in reo. Showing that honestly
   ("no product activity recorded for Jane") is *useful* — it tells the rep
   this person hasn't touched the product, which is itself a signal.
