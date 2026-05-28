# Skill: Daily Traffic Report (PostHog → Slack)

> This skill guides an AI step-by-step to build an agent that automatically pulls daily traffic
> data from PostHog and posts a formatted report to a Slack channel every morning.
> Target users: product and growth teams using SureThing + PostHog + Slack.

---

## Overview

**Suggested Agent Name**: `Daily Traffic Report`

**Agent Description**:
Every morning at [time], automatically pull yesterday's PostHog traffic data — paid channel UTM breakdown, organic source ranking, new vs. returning users, and new signup sources — and post the report to a Slack channel.

**End result**:
- Fires daily at a user-specified time (user's local timezone)
- Report covers: ① Paid channel UTM detail + Google Ads keywords + Meta ad creatives ② Organic sources Top 10 ③ New vs. returning users ④ New signup source breakdown
- Delivered by the Agent's own Slack bot to a specified channel, and also visible in SureThing in-app chat

---

## ⚠️ Execution Constraints (MUST READ)

When executing this skill, you MUST follow these rules strictly:

### Setup Flow (Sequential Gates)

**Steps 1 and 2 are HARD GATES. Do NOT run any data queries until BOTH are completed.**

- Step 1: Connect PostHog → validate with test query → ⛔ STOP and confirm with user
- Step 2: Install Slack bot + pick channel → ⛔ STOP and confirm with user
- ONLY after both gates pass → proceed to Step 3 (data queries) and beyond

If you skip Step 2 and jump straight to data queries after PostHog connects, you are violating this skill's execution contract.

### Report Output Rules

1. **Exactly 5 report modules** — The report MUST contain all 5 query modules defined in Step 3, in order. No more, no less.
2. **No substitutions** — Do NOT replace these HogQL queries with PostHog UI insights, trends API, dashboard screenshots, or your own simplified summary.
3. **No extra sections** — Do NOT add "Top Pages", "Most Active Events", "Session Count", "Bounce Rate", or any metric not specified below.
4. **No generic overviews** — Do NOT summarize the 5 modules into a single paragraph or percentage breakdown. Each module is a separate code-block table.
5. **Empty data = keep header** — If a module returns zero rows (e.g., no paid traffic), still output the section header with "No data for this period". Do NOT skip or hide it.
6. **Use exact SQL** — Copy the HogQL queries from Step 3 verbatim (substituting only the time window and domain variables). Do NOT rewrite, simplify, or "improve" them.

---

## Step 1: Connect PostHog (GATE 1 — must complete before proceeding)

**Do not ask the user for a Project ID.** Connect first, then discover projects automatically.

1. Check `<connected_apps>` in context. If PostHog is not listed as connected:
   - Call `COMPOSIO_MANAGE_CONNECTIONS` with `toolkit: "posthog"` to get an auth URL
   - Send the URL to the user and wait for them to complete the connection
   - You will be notified automatically when the connection is established

2. Once connected, use `COMPOSIO_SEARCH_TOOLS` to find the PostHog "list projects" tool, then call it to fetch all available projects.

3. **Auto-select logic**:
   - If **only one project** is returned → use it silently, no need to ask
   - If **multiple projects** are returned → show the list and ask the user which one to use for the report

4. Confirm connectivity by running a quick test query:
   ```
   SELECT count() as pv FROM events WHERE event = '$pageview' LIMIT 1
   ```
   - If 401 Unauthorized → API key has expired. Ask the user to disconnect and reconnect PostHog.
   - If 504 Gateway Timeout → PostHog is temporarily unavailable; retry in a few minutes.

5. Record the confirmed `project_id` for use in all subsequent queries.

✅ **Gate 1 pass**: Tell user "PostHog connected ✅" then proceed to Step 2.
⛔ **Do NOT proceed** to any data queries until this gate passes.

---

## Step 2: Install Slack Bot + Pick Channel (GATE 2 — must complete before data queries)

> ⚠️ Each SureThing Agent has its own dedicated Slack bot. You **must** use this Agent's bot to
> send the report — not a bot from another Agent or Project.

1. Check if the Agent's Slack bot is installed by calling `slack_channels(operation="list")`.

2. **If NOT installed** (NO_CONNECTED_SLACK_BOT error):
   - Tell the user: "To deliver reports to Slack, I need my Slack bot installed. Go to: project right panel → Agents tab → find this agent → click 'Add to Slack'."
   - ⛔ **STOP HERE** and wait for the user to confirm the bot is installed.

3. **Once bot is confirmed**:
   - List available channels the bot has joined
   - Ask the user which channel should receive the daily report
   - Record the `channel_id` and `integration_id`

4. **Get the Channel ID** (if user provides a name instead of ID):
   In Slack, right-click the channel name → Copy Link. The ID is the `C`-prefixed string at the end of the URL.

✅ **Gate 2 pass**: Tell user "Slack ready ✅ — now I'll generate your first sample report." Then proceed to Step 3.
⛔ **Do NOT run any PostHog data queries** until this gate passes.

---

## Step 3: Write the Data Script

Save the script to `scripts/daily_traffic_report.py`. It accepts one argument `offset_days`
(default `1` = yesterday, `2` = two days ago).

### Time Window Calculation

Convert the user's local "yesterday" into a UTC time range. Read the user's timezone from
`user_context.Timezone` — do not hardcode it.

```python
import sys
from datetime import datetime, timedelta

# Set UTC_OFFSET from user's timezone (e.g., Asia/Shanghai = 8, UTC = 0, America/New_York = -5)
UTC_OFFSET = <UTC_OFFSET_FROM_USER_TIMEZONE>

def get_window(offset_days=1):
    now_utc = datetime.utcnow()
    now_local = now_utc + timedelta(hours=UTC_OFFSET)
    target_local = now_local.date() - timedelta(days=offset_days)

    start_utc = datetime(target_local.year, target_local.month, target_local.day) \
                - timedelta(hours=UTC_OFFSET)
    end_utc = start_utc + timedelta(days=1)

    label = target_local.strftime('%Y-%m-%d')
    return label, start_utc.strftime('%Y-%m-%d %H:%M:%S'), end_utc.strftime('%Y-%m-%d %H:%M:%S')
```

### PostHog Query Helper

All queries use `run_composio_tool` with `kind: HogQLQuery`:

```python
PROJECT_ID = '<CONFIRMED_PROJECT_ID>'  # set after Step 1

def query(sql):
    data, error = run_composio_tool('POSTHOG_CREATE_QUERY_IN_PROJECT_BY_ID', {
        'project_id': PROJECT_ID,
        'query': {'kind': 'HogQLQuery', 'query': sql}
    })
    if error:
        print(f'[WARN] query error: {str(error)[:200]}', file=sys.stderr)
        return [], []

    if isinstance(data, dict):
        if 'results' in data:
            return data['results'] or [], data.get('columns', [])
        inner = data.get('data') or data.get('response', {}).get('data', {}) or {}
        if 'results' in inner:
            return inner['results'] or [], inner.get('columns', [])
    return [], []
```

### ⚠️ Critical: PostHog Field Naming

UTM parameters in HogQL do **not** have a `$` prefix — unlike other auto-captured properties.
Getting this wrong causes all paid traffic to appear as zero.

| Field type | Correct | Wrong |
|---|---|---|
| UTM params | `properties.utm_medium` | `properties.$utm_medium` ❌ |
| UTM params | `properties.utm_source` | `properties.$utm_source` ❌ |
| Auto-captured | `properties.$referring_domain` | `properties.referring_domain` ❌ |
| Person initial UTM | `person.properties.$initial_utm_source` | `person.properties.initial_utm_source` ❌ |

### The Five Core Queries

**① Overall PV / UV**
```sql
SELECT count() as pv, count(distinct distinct_id) as uv
FROM events
WHERE event = '$pageview'
  AND timestamp >= '{start_utc}' AND timestamp < '{end_utc}'
```

**② Paid Traffic (UTM-tagged)**
```sql
SELECT
    properties.utm_source   as utm_source,
    properties.utm_medium   as utm_medium,
    properties.utm_campaign as utm_campaign,
    properties.utm_term     as utm_term,
    properties.utm_content  as utm_content,
    count()                 as pv,
    count(distinct distinct_id) as uv
FROM events
WHERE event = '$pageview'
  AND timestamp >= '{start_utc}' AND timestamp < '{end_utc}'
  AND properties.utm_medium != ''
  AND properties.utm_medium IS NOT NULL
GROUP BY utm_source, utm_medium, utm_campaign, utm_term, utm_content
ORDER BY pv DESC
LIMIT 100
```

Post-processing from these results:
- **Google Ads keywords**: filter where `utm_source = 'google'`, use `utm_term` field, Top 10 by PV
- **Meta ad creatives**: filter where `utm_source IN ('meta', 'fb', 'facebook')` AND `utm_medium IN ('paid_social', 'paid')`, use `utm_term` field, Top 10 by PV

**③ Organic Sources (no UTM)**
```sql
SELECT
    properties.$referring_domain as referring_domain,
    count()                      as pv,
    count(distinct distinct_id)  as uv
FROM events
WHERE event = '$pageview'
  AND timestamp >= '{start_utc}' AND timestamp < '{end_utc}'
  AND (properties.utm_medium = '' OR properties.utm_medium IS NULL)
  AND properties.$referring_domain != ''
  AND properties.$referring_domain IS NOT NULL
  AND properties.$referring_domain NOT LIKE '%<YOUR_PRODUCT_DOMAIN>%'
GROUP BY referring_domain
ORDER BY pv DESC
LIMIT 10
```

Replace `<YOUR_PRODUCT_DOMAIN>` with the product's actual domain to filter internal navigation.

**④ New vs. Returning Users**
```sql
SELECT
    if(person.created_at >= '{start_utc}' AND person.created_at < '{end_utc}',
       'New', 'Returning') as user_type,
    count()                       as pv,
    count(distinct distinct_id)   as uv
FROM events
WHERE event = '$pageview'
  AND timestamp >= '{start_utc}' AND timestamp < '{end_utc}'
GROUP BY user_type
ORDER BY uv DESC
```

**⑤ New Signup Source Breakdown**
```sql
SELECT
    coalesce(person.properties.$initial_utm_source, '(direct)') as utm_source,
    coalesce(person.properties.$initial_utm_medium, '(none)')   as utm_medium,
    count(distinct distinct_id) as registrations
FROM events
WHERE event = '<YOUR_SIGNUP_EVENT>'
  AND timestamp >= '{start_utc}' AND timestamp < '{end_utc}'
GROUP BY utm_source, utm_medium
ORDER BY registrations DESC
LIMIT 15
```

Replace `<YOUR_SIGNUP_EVENT>` with the actual event name used in PostHog (e.g. `user_created`,
`sign_up`). Auto-detect by querying PostHog event definitions for common signup event names.

---

## Step 4: Slack Formatting

### Why not Markdown tables?

Slack does **not** render `| col | col |` Markdown tables — the pipe characters appear as literal
text. The correct approach is a monospace **code block** with aligned columns.

### CJK character width

If the report contains Chinese, Japanese, or Korean characters, standard `str.ljust()` breaks
alignment because CJK characters occupy 2 columns in monospace fonts. Use `unicodedata` to
calculate the true display width:

```python
import unicodedata

def cw(s):
    """Return display width (CJK chars count as 2)."""
    w = 0
    for c in str(s):
        if unicodedata.east_asian_width(c) in ('W', 'F'):
            w += 2
        else:
            w += 1
    return w

def slack_table(headers, rows, right_cols=None):
    """
    Render a Slack code-block table with aligned columns.
    right_cols: set of column indices to right-align (recommended for numbers).
    """
    if right_cols is None:
        right_cols = set()
    all_data = [list(headers)] + [list(r) for r in rows]
    col_widths = [max(cw(str(row[i])) for row in all_data) for i in range(len(headers))]

    def fmt_row(row):
        cells = []
        for i, cell in enumerate(row):
            s = str(cell)
            pad = col_widths[i] - cw(s)
            cells.append((' ' * max(0, pad) + s) if i in right_cols else (s + ' ' * max(0, pad)))
        return '  '.join(cells).rstrip()

    lines = ['```']
    lines.append(fmt_row(headers))
    lines.append('  '.join('─' * col_widths[i] for i in range(len(headers))))
    for row in rows:
        lines.append(fmt_row(row))
    lines.append('```')
    return '\n'.join(lines)
```

### Formatting rules

- ❌ Do not use `*bold*` or `_italic_` mrkdwn inside code blocks, or around CJK text — Slack
  cannot detect the closing marker after CJK punctuation and renders literal asterisks
- ✅ Use plain text + emoji for visual structure (`📊 ① ② ③ ④ 🆕 🔁` render correctly everywhere)
- ✅ Right-align numeric columns, left-align text columns

### Report structure

```
Daily Traffic Report · {date_label}
📊 Overview — PV {x,xxx} / UV {xxx} / Pages/user {x.x}

① Paid Channels (UTM-tagged)
{slack_table: Channel | PV | UV}

Google Ads Keywords Top 10
{slack_table: Keyword | PV | UV}

Meta Ad Creatives Top 10
{slack_table: Creative (utm_term) | PV | UV}

② Organic Sources Top 10 (no UTM)
{slack_table: Source | PV | UV}

③ New vs. Returning Users
{slack_table: Type | PV | UV | Pages/user}

④ New Signups Yesterday: {n}
{slack_table: Source | Signups}
```

---

## Step 5: Run Once Manually to Verify

Before setting up the scheduled task, run the script manually:

```bash
python scripts/daily_traffic_report.py 1   # 1 = yesterday, 2 = two days ago
```

Verification checklist:
- [ ] Paid channel data is non-empty (if the product runs paid ads)
- [ ] Google Ads keyword data comes from `utm_term`, numbers look reasonable
- [ ] Organic sources — Top 1 is typically `$direct` (direct / unknown)
- [ ] New + Returning UV roughly equals total UV (small variance is normal)
- [ ] New signups count matches the product's backend dashboard (within ~5%)

If paid data is all zeros → first check that `utm_medium` has no `$` prefix in the query.

---

## Step 6: Sample Preview (do NOT skip)

Before setting up the scheduled task, show the user the full formatted report in chat:

- Date range covered
- All 5 sections rendered as Slack code blocks
- Ask: "Report look good? If yes, I'll set up the daily cron and start posting to your Slack channel."

Only proceed to Step 7 when user explicitly approves.

---

## Step 7: Send the Report to Slack

Use the `slack_channels` tool — **not** Composio `SLACKBOT_*` or `SLACK_*` tools, which bypass
the Agent's own bot and send from the wrong account:

```python
slack_channels(
    operation      = "send",
    channel_id     = "<CHANNEL_ID>",       # from Step 2
    integration_id = "<INTEGRATION_ID>",   # from Step 2
    text           = "<report_body>"
)
```

---

## Step 8: Create the Scheduled Task

Once manual verification passes, create a daily cron task:

```
task_manage(
    operation = "create",
    title     = "Daily Traffic Report — PostHog UTM & Source Analysis",
    executor  = "ai",
    trigger   = {
        type   : "cron",
        config : {
            expression : "0 9 * * *",        # 09:00 daily — adjust as needed
            timezone   : "<USER_TIMEZONE>"   # e.g. "Asia/Shanghai", "America/New_York"
        }
    },
    action    = """
        Run daily at [time] to pull yesterday's traffic data from PostHog and post the report.

        Steps:
        1. Run script: scripts/daily_traffic_report.py with argument 1 (yesterday)
        2. Use the script stdout as the report body
        3. Call slack_channels(operation="send", channel_id="<CHANNEL_ID>",
           integration_id="<INTEGRATION_ID>", text=<report_body>)
        4. If PostHog returns a 504 timeout, wait 30 minutes and retry once

        Script: scripts/daily_traffic_report.py
    """,
    result_sinks = {
        slack : {
            integration_id : "<INTEGRATION_ID>",
            channels       : [{ channel_id: "<CHANNEL_ID>", channel_name: "<CHANNEL_NAME>" }]
        }
    }
)
```

---

## Step 9: End-to-End Verification

Manually trigger the task once immediately after creation, and confirm:

- [ ] Script runs successfully, no 401 / 504 errors
- [ ] Report is sent by **this Agent's own bot** (not another Project's bot)
- [ ] Slack code block renders correctly — columns are aligned, no garbled characters
- [ ] SureThing in-app chat also receives the report
- [ ] Scheduled task status is `pending`, next trigger time is correct

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| All paid channel data is zero | Query uses `$utm_medium` instead of `utm_medium` | Remove the `$` prefix |
| PostHog returns 401 | API key expired | Connected Apps → PostHog → disconnect and reconnect |
| Slack shows raw pipe characters | Using Markdown table syntax | Switch to code block + CJK-aware column padding |
| Report sent by the wrong bot | Wrong `integration_id` used | Call `slack_channels(list)` to confirm this Agent's `integration_id` |
| CJK columns misaligned | Using `str.ljust()` without CJK width awareness | Use `unicodedata.east_asian_width` to compute display width |
| New signup count is zero | Wrong event name in query | Auto-detect from PostHog event definitions or confirm with dev team |
| PostHog 504 timeout | Transient DB maintenance window (often around 01:00 local time) | Add a 30-minute retry in the task action |