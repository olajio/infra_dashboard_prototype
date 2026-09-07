# Offshore Team Update — MIM Dashboard v2

Subject: **MIM Dashboard v2 — action items applied; please replicate on your copy**

---

Hi team,

Following the review meeting, I've applied the 15 action items from *MIM Dashboard Updated Action Items.docx* to the enhanced dashboard. The updated file (`mim-dashboard-enhanced.ndjson`) is on the `main` branch of the repo and can be imported directly via Stack Management → Saved Objects → Import (overwrite).

I need you to replicate the same changes on your working copy so we stay in sync. The changes and the ES|QL for each are below — starting point is the previous version of `mim-dashboard-enhanced.ndjson`.

---

## What changed (15 items)

### Removed
- **Business Service Impact panel** — deleted (Row 5 in the old dashboard).

### Converted from table to pie chart
- **Repeat Offenders** — now a pie of CIs with >1 P1/P2 incident (top 15 slices)
- **Detection Source** — now a pie by `contact_type`
- **Assignment Group Load** — now a pie by `assignment_group.name`

### New panels
- **Alert vs Auto-Generated Incidents** (pie, between Detection Source and Top Affected CIs)
- **SLA Breached / At-Risk / Within** (3 metric tiles) — assumes P1 = 4h, P2 = 8h SLA. Adjust the CASE expression if your SLA agreement differs.
- **SLA Breach Detail table** (full-width, below SLA tiles) — every column requested in item #15
- **Active P1/P2 — Pending Action table** (replaces the old ageing-only table; adds Support Group and Status columns; uses CI Name not CI Number)

### Modified queries
- **MTTR Trend** — now split into two series (P1 vs P2) instead of a single combined line
- **Incidents by CI / Support Team** — restricted to P1+P2 only (P3 column dropped, WHERE clause added)
- **Closure Rate tooltip** — clarified: numerator = all resolved P1/P2 in window; denominator = all P1/P2 opened in the same window

### Not changed
- Open Incidents / Aging Analysis (item #10, #11)
- Recurrence Rate tile + repeat-offender pair already act as a drill-down couple (item #6)
- Line-chart drill-down (item #3) — Kibana Lens enables click-to-filter by default; no extra configuration needed. Clicking a data point on a line chart or a table row filters the whole dashboard.

---

## ES|QL to paste

Below is the ES|QL for every new or modified panel. Add these in **Lens → ES|QL mode**, choose the visualization type noted, then map the columns.

### 1. Active P1/P2 — Pending Action (datatable)
```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2) AND resolved_at IS NULL
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| EVAL status_val = "Open (pending action)"
| KEEP number, ci.name, priority, status_val, ci.support_group.l2.name,
       assignment_group.name, age_hours, opened_at
| SORT age_hours DESC
| LIMIT 100
```

### 2. MTTR Trend split by P1 vs P2 (line, multi-series)
```esql
FROM servicenow-incidents-*
| WHERE opened_at IS NOT NULL AND resolved_at IS NOT NULL AND priority IN (1, 2)
| EVAL mins = DATE_DIFF("minutes", opened_at, resolved_at)
| WHERE mins > 0
| STATS median_mins = MEDIAN(mins) BY bucket = DATE_TRUNC(1 day, @timestamp), priority
| EVAL metric_val = ROUND(median_mins / 60.0, 1)
| SORT bucket ASC
```
X-axis: `bucket`. Y-axis: `metric_val`. Split series: `priority`.

### 3. Incidents by CI / Support Team — P1+P2 only (datatable)
```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS
    p1 = COUNT(CASE(priority == 1, number, null)),
    p2 = COUNT(CASE(priority == 2, number, null)),
    total_val = COUNT(*)
  BY ci_name = ci.name, team = ci.support_group.l2.name
| SORT total_val DESC
| LIMIT 50
```

### 4. Repeat Offenders (pie)
```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS incidents = COUNT(*) BY ci_name = ci.name
| WHERE incidents > 1
| SORT incidents DESC
| LIMIT 15
```
Slice: `ci_name`. Value: `incidents`.

### 5. Detection Source (pie)
```esql
FROM servicenow-incidents-*
| WHERE contact_type IS NOT NULL
| STATS incidents = COUNT(*) BY contact_type
| SORT incidents DESC
| LIMIT 10
```
Slice: `contact_type`. Value: `incidents`.

### 6. Alert vs Auto-Generated (pie)
```esql
FROM servicenow-incidents-*
| WHERE contact_type IN ("alert", "auto_generated")
| STATS incidents = COUNT(*) BY contact_type
| SORT incidents DESC
```
Slice: `contact_type`. Value: `incidents`.

### 7. Assignment Group Load (pie)
```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2) AND assignment_group.name IS NOT NULL
| STATS incidents = COUNT(*) BY team = assignment_group.name
| SORT incidents DESC
| LIMIT 15
```
Slice: `team`. Value: `incidents`.

### 8. SLA Breached (metric tile)
```esql
FROM servicenow-incidents-*
| WHERE resolved_at IS NULL AND priority IN (1, 2)
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| EVAL sla_target = CASE(priority == 1, 4, priority == 2, 8, 999)
| WHERE age_hours >= sla_target
| STATS metric_val = COUNT(*)
```
Format: Number, pattern `0,0`.

### 9. SLA At Risk (metric tile)
```esql
FROM servicenow-incidents-*
| WHERE resolved_at IS NULL AND priority IN (1, 2)
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| EVAL sla_target = CASE(priority == 1, 4, priority == 2, 8, 999)
| WHERE age_hours >= sla_target * 0.8 AND age_hours < sla_target
| STATS metric_val = COUNT(*)
```

### 10. SLA Within Target (metric tile)
```esql
FROM servicenow-incidents-*
| WHERE resolved_at IS NULL AND priority IN (1, 2)
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| EVAL sla_target = CASE(priority == 1, 4, priority == 2, 8, 999)
| WHERE age_hours < sla_target * 0.8
| STATS metric_val = COUNT(*)
```

### 11. SLA Breach Detail (datatable)
```esql
FROM servicenow-incidents-*
| WHERE resolved_at IS NULL AND priority IN (1, 2)
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| EVAL sla_target_hrs = CASE(priority == 1, 4, priority == 2, 8, 999)
| EVAL sla_status = CASE(
      age_hours >= sla_target_hrs, "Breached",
      age_hours >= sla_target_hrs * 0.8, "At Risk",
      "Within SLA")
| EVAL time_delta_hrs = age_hours - sla_target_hrs
| EVAL state_val = "Open"
| KEEP number, priority, state_val, assignment_group.name, ci.name,
       age_hours, sla_target_hrs, sla_status, time_delta_hrs
| SORT age_hours DESC
| LIMIT 100
```
Columns render as: Incident # | Priority | State | Assigned Group | CI Name | Ticket Age (hrs) | SLA Target (hrs) | SLA Status | Time Overdue (+) / Left (−).

---

## Notes

- **SLA targets (P1=4h, P2=8h) are ITIL defaults.** Update the `CASE(priority == 1, 4, priority == 2, 8, 999)` expression in every SLA panel to match your actual SLA agreement.
- **Alert vs Auto-Generated origin field**: the split uses `contact_type` values already present in the index (`alert`, `auto_generated`). If a dedicated `source` field becomes available, extend the WHERE clause.
- **Drill-down (item #3)**: Kibana Lens has default click-to-filter on all charts and tables. Clicking a pie slice, table row, or line-chart point applies a filter that all other panels respond to. No extra configuration required. The dashboard header markdown notes this behaviour so users know to click.

If anything doesn't render as expected on import, the query column formats are documented in `docs/mim-dashboard-instructions.md`. Ping me if you hit a data-mapping issue.

Thanks,
[Your name]

---

*Attachments: `mim-dashboard-enhanced.ndjson`, `MIM_KPI_updated.xlsx`, `docs/mim-dashboard-instructions.md`*
