# Behavioral opportunity agent

Weekly Databricks job for Scenario 3. Nobody is in a conversation. The agent reads aggregate store behaviour, decides whether a promotion is worth designing, and - only when it is - asks Vincent to draft cohorts. A marketer arrives to a thread that already contains both agents' reasoning.

The pitch is in [HACKATHON.md §7](../../HACKATHON.md#7-scenario-3--weekly-opportunity-agent-to-vincent). This folder is the Databricks source: why the decision is shaped this way, the two jobs, the tools, and the exact handoff. Vincent's implementation lives in the Vincent repository.

## Why it works this way

The commercial case is purchase intent that holds up or grows while conversion falls: people keep browsing, carting, and checking out, and fewer of them buy than the longer history would suggest. A one-week spike is not enough. Value sitting in carts or at checkout is value at stake, not revenue a discount will recover. The agent is told not to invent causes it cannot see (seasonality, competitors, expected uplift) and not to add overlapping product or cart figures into one total.

`NO_ALERT` is a successful run. The agent is expected to send nothing when the evidence is thin. A job that briefed Vincent every Monday would be a report generator.

Privacy is a hard boundary, not a prompt courtesy. The daily table, both tools, and the brief contain thresholds, counts, rates, products, devices, and amounts. No emails, no customer ids, no raw events, no list of people. The agent describes who *in aggregate* and asks Vincent to propose cohorts. It does not pick the audience.

## The two jobs

Both are serverless Databricks Workflows. The notebooks in this folder are their source.

| | Daily report | Weekly review |
|---|---|---|
| Notebook | `databricks_behavioral_reports.py` | `databricks_behavioral_agent_run.py` |
| Job id | `451694092272892` | `590707914570366` |
| Schedule | every day at 23:30 `Europe/Warsaw`, and **Run now** | Mondays 09:00 `Europe/Warsaw`, and **Run now** |
| Writes | `databricks-hackathon.voucherify_otto.behavioral_report_daily` | `databricks-hackathon.voucherify_otto.behavioral_agent_runs` |

**Daily report.** Reads `customer_events_live` and `customers_live`. For the 7-, 30-, 90-, and 180-day windows it computes viewers and views, high-intent customers (5+ product views, or any cart update, or any checkout), how many of those bought nothing, conversion, cart and checkout abandoners, purchase revenue, abandoned cart and checkout value, the desktop/mobile split, the products high-intent nonbuyers look at most, and aggregate K-means cluster profiles. One row per UTC report date. No identifiers.

Defaults are `seed_history=false` and `seed_future_days=0`, so a normal run only upserts today's row. The demo table also holds 180 deterministic historical days and 14 future days, produced only when those parameters are set. The story those rows converge on: 30-day viewers rise from about 842 to 1,055, high-intent conversion falls from about 84.5% to 69.7%, high-intent nonbuyers grow from about 130 to 318, and abandoned checkout value is about £145k. The agent tools and the weekly notebook both ignore rows with `report_date` after today, so a future seed stays invisible until that date.

**Weekly review.** Does no analysis. It calls the Supervisor's serving endpoint, then acts as transport and audit.

- `NO_ALERT` - nothing is posted. The decision is still appended to `behavioral_agent_runs`.
- A brief starting with `NOTIFY_VINCENT:` - the prefix is stripped and the Markdown is posted to Vincent. The full agent response and Vincent's HTTP status are appended to the same audit table. A transport failure is written first, then the notebook raises.

## The agent and its tools

Agent Bricks Supervisor, display name **Behavioral Opportunity Analyst**.

| | |
|---|---|
| Supervisor id | `fe23f01a-9cf0-4e36-8f8e-0bc376bb3d08` |
| Serving endpoint | `mas-fe23f01a-endpoint` |
| Instructions | `instructions.md` - pasted into Agent Bricks |
| Tools | `tools.sql` - create or replace both functions, then attach them |

The instructions require both calls before a decision. The two results are different evidence:

- `get_behavioral_opportunity_snapshot()` - the latest report with `report_date <= today`. The 7/30/90/180 figures are overlapping lookbacks that all end today. A healthier 180-day conversion than 30-day is not "conversion six months ago". Also returns product affinity, device mix, aggregate cluster profiles, and the only segment labels Vincent can target.
- `get_behavioral_opportunity_trend()` - the historical series: dated checkpoints and the change versus 30, 90, and 180 days ago. Persistence comes from here, not from comparing the overlapping snapshot windows.

Each payload stays under the Agent Bricks 4,000-character tool-output limit.

The snapshot's `targetable_segments` are the five labels synchronised onto Bloomreach profiles (`databricks_behavior_segment`), from the current export of 1,111 customers:

| Label | Customers |
|---|---|
| Frequent buyers | 391 |
| Light-engagement customers | 284 |
| Higher-value customers | 178 |
| Cart-heavy low-frequency shoppers | 151 |
| High-intent browsers | 107 |

The brief may use those labels, in ordinary marketing language, plus behaviour Vincent can already filter on: product views, cart updates, checkouts, purchase frequency or recency, device, or an exact product title. It must not mention the property name, cluster ids, table names, or EQL. Product categories are not available, so the brief does not propose category targeting.

An alert is one Markdown message that starts with `NOTIFY_VINCENT:` and contains a headline, who to target, why these customers, what it is worth (GBP), why a campaign could pay including the cannibalisation risk, and a request that Vincent propose several cohorts and not pick the audience.

## Event id

```text
behavioral-opportunity-<report-date>-<job-run-id>
```

`<report-date>` is the latest report row on or before today. `<job-run-id>` is the Workflow run that produced the brief, so every run that sends a brief opens its own thread in Vincent.

The posted body, to `POST /api/v1/internal/proactive/signals`:

```json
{
  "event_id": "behavioral-opportunity-<report-date>-<job-run-id>",
  "tenant_id": "<organisation>",
  "project_id": "<project>",
  "type": "behavioral_opportunity",
  "message": "The Markdown brief, without the NOTIFY_VINCENT: prefix."
}
```

## What Vincent does with it

Vincent reads only `message`. The rest of the body tells it which organisation and project the brief belongs to.

- The brief becomes the first visible turn of a new organisation-wide thread, attributed to the Databricks agent.
- Vincent's marketing node runs in exploratory mode. It turns the described characteristics into concrete cohort candidates and sizes each one on Bloomreach, so the numbers are real.
- It replies in the same thread and stops at the `cohort_selection` gate. It picks no audience and creates nothing in Voucherify.
- The thread waits unclaimed until a marketer claims it, chooses a cohort, and continues the conversation.

This folder does not create campaigns, pick a cohort, or write to Voucherify.
