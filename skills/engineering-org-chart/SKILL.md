---
name: engineering-org-chart
description: >
  Builds a visual, top-to-bottom engineering org chart for any target account using reo
  open data (LinkedIn profiles) with reo activity signals overlaid on every node.
  Shows the full engineering hierarchy — CTO/VP Eng → Directors → Managers → Senior ICs → ICs —
  grouped by sub-team (Platform, Data/ML, Backend, Frontend, DevOps/SRE, Security).
  Signal-active employees are highlighted green; stale signals are amber; no signals are grey.
  Output is always an interactive HTML tree diagram rendered inline via show_widget.

  Use this skill whenever the user asks for:
  "engineering org chart for [company]", "map the eng team at [company]",
  "show me the engineering hierarchy at [company]", "who leads engineering at [company]",
  "top to bottom eng team for [company]", "who are the engineers at [company]",
  "build me an org chart", or any request to visualise, map, or understand the engineering
  structure at a named account. Also trigger when researching a company's engineering team
  size, leadership, or sub-team breakdown.

compatibility: "Requires reo-gateway tools: reo_list_prospects, reo_get_tech_function_job_counts, reo_get_account_activities. Requires visualize:show_widget."
---

# Engineering Org Chart — Skill Instructions

## What this skill does
Maps the full engineering hierarchy at a target company using two data layers:
- **Layer 1 (Prospects)**: `reo_list_prospects` filtered to engineering tech_functions — authoritative people list
- **Layer 2 (Signals)**: `reo_get_account_activities` — who has actually touched your product and when
- **Output**: Interactive HTML tree chart, nodes coloured by signal status, grouped by sub-team

⚠️ CRITICAL DATA SOURCING RULE:
The people list comes ENTIRELY from `reo_list_prospects`. Do NOT derive the people list from
activity signals. Signals tell you WHO IS ACTIVE, not WHO EXISTS.
Using activity data to discover people will surface whoever uses the product (often the
customer's sales/GTM team), NOT the engineering org you want to map.

---

## CONFIG

SIGNAL_ACTIVE_DAYS: 30
# Node is GREEN 🟢 if they have an activity signal within this many days.
# Node is AMBER 🟡 if a signal exists but is older than SIGNAL_ACTIVE_DAYS.
# Node is GREY ⚫ if no signals at all.

SIGNAL_LOOKBACK_DAYS: 90

SHOW_LINKEDIN_LINKS: true

---

## Step 1 — Resolve Company Domain

If the user gives a company name, infer the domain (e.g. "Notion" → "notion.so").
If ambiguous, ask the user to confirm.

---

## Step 2 — Fetch Team Headcount (reo_get_tech_function_job_counts)

Call `reo_get_tech_function_job_counts` with the domain.
Returns headcount by tech_function. Use to:
- Understand team composition before querying people
- Show totals in the chart header
- Skip sub-teams with count = 0 (no call needed for empty teams)

The valid tech_function values from this API are:
- "Engineering Leadership"
- "Artificial Intelligence & Machine Learning"
- "Backend Engineering"
- "Cloud & Infrastructure"
- "Full Stack"
- "Frontend Engineering"
- "DevOps Platform & Reliability"
- "Security & Compliance"
- "QA Automation & Testing"
- "Infrastructure & IT Operations"
- "Mobile Engineering"

---

## Step 3 — Fetch All Engineering People (reo_list_prospects)

⚠️ THIS IS THE ONLY SOURCE OF PEOPLE. Not activity data. Not open data profiles.

Call `reo_list_prospects` with:
- `domain`: the resolved company domain
- `tech_function`: ALL engineering functions from Step 2 that have count > 0 (see list above)
- `page_size`: 50

Check `pagination.total_pages` in the response. If > 1, make additional calls with
`page_number: 2`, `page_number: 3`, etc. until all pages are fetched.

**Do not filter by seniority** — fetch all levels so you can build the full hierarchy.

For each person capture:
- `id` — LinkedIn public_id (used for URL construction and signal matching)
- `full_name`
- `designation` — job title
- `seniority` — reo seniority label (use directly, don't re-infer from title)
- `tech_function` — array, used to assign sub-team
- `linkedin` — full LinkedIn URL
- `is_likely_to_engage` — show ⚡ badge if true
- `current_position_start_date`
- `state` — US state or equivalent region
- `country` — country name

---

## Step 4 — Classify: Hierarchy Level + Sub-Team

### 4a. Hierarchy Level — use reo `seniority` field directly

| reo seniority | Level | Display label |
|---------------|-------|---------------|
| C-Level | 0 | 👑 C-Suite |
| VP | 1 | 🏆 VP / Head |
| Director | 2 | 🎯 Director / Principal |
| Manager | 3 | 🧩 Manager / Staff |
| Senior | 4 | ⭐ Senior / Lead |
| Mid | 5 | 💻 Mid-level IC |
| Junior | 6 | 🌱 Junior |
| null / unknown | 5 | 💻 IC |

### 4b. Sub-Team — map directly from `tech_function` array

| reo tech_function | Sub-Team |
|-------------------|----------|
| Engineering Leadership | 🏛️ Engineering Leadership |
| Artificial Intelligence & Machine Learning | 🤖 AI / ML |
| Backend Engineering | 🖥️ Backend |
| Cloud & Infrastructure | ☁️ Cloud & Infra |
| Full Stack | 🔧 Full Stack |
| Frontend Engineering | 🎨 Frontend |
| Mobile Engineering | 📱 Mobile |
| DevOps Platform & Reliability | ⚙️ DevOps / Platform |
| Security & Compliance | 🔒 Security |
| QA Automation & Testing | 🧪 QA / Testing |
| Infrastructure & IT Operations | 🛠️ Infra & IT |

If `tech_function` is null/empty, fall back to title keywords (see Appendix at bottom).

### 4c. Location Group — derive from `state` + `country` fields

Build a `location_group` label for each person:
```
if country == "United States" and state is not null → location_group = state  (e.g. "California")
elif country is not null                            → location_group = country (e.g. "India")
else                                                → location_group = "Unknown"
```

After processing all people, consolidate US states with < 2 people into "Other US" to keep
the chart readable if there are many distinct states.


---

## Step 5 — Fetch Activity Signals (reo_get_account_activities)

Call `reo_get_account_activities` with the account ID (resolve domain → UUID via account lookup first if needed).

For each activity event extract: person identifier, activity_type, timestamp.

### Build signal map: prospect id → signal status
```
For each person from Step 3:
  If matched to activity AND last signal within SIGNAL_ACTIVE_DAYS  → 🟢 "active"
  If matched to activity AND last signal older than SIGNAL_ACTIVE_DAYS → 🟡 "stale"
  No match → ⚫ "none"
  Store: last_seen date, total event count, top pages visited
```

**If activity data returns people NOT in the engineering prospects list** (e.g. sales/GTM users):
- Include them in the signal map but do NOT add them to org chart nodes
- Collect their names/titles for a footnote below the chart

---

## Step 6 — Render the HTML Org Chart (show_widget)

Use `show_widget`. Always render as HTML — never output a markdown table.

### Color palette
- Page bg: `#111827`; Panel bg: `#1f2937`; Node bg: `#374151`
- Active border/dot: `#4ade80`; Stale: `#f59e0b`; None: `#4b5563` / `#6b7280`
- Text primary: `#f9fafb`; secondary: `#9ca3af`; muted: `#6b7280`
- Sub-team header: `#1e3a5f` bg; Connector lines: `#374151`
- Accent: `#60a5fa` blue, `#a78bfa` purple

### Layout — Two View Modes with Toggle

Add a **toggle bar** in the header with two buttons: `[ By Team ]  [ By Location ]`
Default view is **By Team**. Clicking the toggle switches the swimlane dimension without
reloading — use JavaScript to show/hide the two pre-rendered views.

**View A — By Team (default)**
```
┌─────────────────────────────────────────────────────────┐
│ HEADER: [Company] Engineering Org                        │
│ [N] engineers · 🟢 [N] active · 🟡 [N] stale · ⚫ [N]  │
│ [ By Team ▼ ]  [ By Location ]    Legend: 🟢 🟡 ⚫      │
├─────────────────────────────────────────────────────────┤
│         👑 C-Suite & 🏆 VP Eng  (full-width row)         │
│         [NODE]  [NODE]  ...                              │
├──────────┬──────────┬─────────┬──────────┬──────────────┤
│ 🤖 AI/ML │🖥️ Backend│☁️ Cloud │🎨 Frontend│⚙️ DevOps ... │
│ [L2–L6 nodes sorted by seniority within each column]    │
└──────────┴──────────┴─────────┴──────────┴──────────────┘
```

**View B — By Location**
```
┌─────────────────────────────────────────────────────────┐
│ HEADER (same)                                            │
│ [ By Team ]  [ By Location ▼ ]    Legend: 🟢 🟡 ⚫      │
├────────────────┬───────────────┬──────────┬─────────────┤
│ 📍 California  │ 📍 New York   │ 📍 India │ 📍 Other... │
│ [N people]     │ [N people]    │ [N]      │             │
│                │               │          │             │
│ Grouped by     │ Grouped by    │ Grouped  │             │
│ sub-team       │ sub-team      │ by team  │             │
│ within col     │ within col    │          │             │
│                │               │          │             │
│ [NODE cards,   │ [NODE cards]  │ [NODES]  │             │
│  sorted by     │               │          │             │
│  seniority]    │               │          │             │
└────────────────┴───────────────┴──────────┴─────────────┘
```

In the **By Location** view:
- Each column = one `location_group` (state for US, country for international)
- Show the location flag emoji if country is known (🇺🇸 🇬🇧 🇮🇳 etc.)
- Within each column, group nodes by sub-team label (small grey sub-header), then by seniority
- Show "📍 [Location] · [N] engineers" as the column header
- Sort columns by headcount descending (largest location first)

### Node anatomy
Each card has:
- Coloured left border (green / amber / grey)
- Initials avatar (bg matches signal colour)
- Full name (links to LinkedIn if SHOW_LINKEDIN_LINKS = true)
- Job title (11px muted)
- Signal line: dot + "Last seen [date]" or "No signals"
- ⚡ badge if `is_likely_to_engage = true`

### Interactivity
- **Click node** → tooltip: pages visited, activity types, event count, first/last seen
- **Sub-team column headers** → collapse/expand
- **Hover** → lift shadow
- If sub-team has > 8 ICs: show first 5 + "Show +N more" chip

### Non-engineering activity note (if any)
If Step 5 found active users not in the engineering org:
> "⚠️ [N] non-engineering users also active: [names / titles]. Likely internal Reo product users."

### 🧠 What this tells us
2–3 sentences:
- Which sub-team has the most signal activity
- Which seniority level is engaged (ICs vs leadership)
- Notable gaps (e.g. "No signal from Engineering Leadership — likely bottom-up evaluation")

### 🎯 Suggested next steps
3 prioritised actions tied to specific people or gaps:
- 🔴 Urgent | 🟡 This month | ⚪ When ready

---

## Edge Cases

| Situation | Handling |
|-----------|----------|
| No engineers from reo_list_prospects | Empty state: "No engineering profiles found for [domain]." Show headcount from Step 2. |
| No signals | Full chart with all ⚫ nodes. Banner: "No product activity — use as prospecting map." |
| Domain can't be resolved | Ask user to confirm domain. |
| Large team (50+ engineers) | Collapse ICs by default with "Show all" per sub-team. |
| Only 1 active engineer | Banner: "⚠️ Single-threaded — high stall risk." |
| null tech_function | Fall back to title keyword sub-team matching (Appendix). |

---

## Appendix — Title Keyword Fallback for Sub-Team (when tech_function is null)

| Sub-Team | Title keywords |
|----------|---------------|
| 🤖 AI / ML | ML Engineer, Machine Learning, AI Engineer, Data Scientist, NLP, LLM, MLOps |
| 🖥️ Backend | Backend Engineer, API Engineer, Server-side, Golang, Python Eng |
| ☁️ Cloud & Infra | Infrastructure, Cloud Engineer, SRE, Site Reliability |
| ⚙️ DevOps | DevOps, Platform Engineer, Platform Reliability |
| 🔒 Security | Security Engineer, AppSec, InfoSec, CISO |
| 🎨 Frontend | Frontend, UI Engineer, React, Vue, Angular |
| 📱 Mobile | iOS, Android, Mobile, React Native |
| 🔧 Full Stack | Full Stack, Fullstack |
| 🏛️ Engineering Leadership | CTO, VP Eng, Director of Eng, Head of Eng, Engineering Manager |
