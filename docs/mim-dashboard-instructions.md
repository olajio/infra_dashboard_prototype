# MIM Dashboard — Consultant Build Instructions (v2)

Audience: offshore Kibana engineers building the enhanced MIM dashboard.  
Source index: `servicenow-incidents-*`  
Kibana version: 8.x (tested at 8.8+)

**v2 update:** incorporates the 15 action items from *MIM Dashboard Updated Action Items.docx*. Dashboard is now 21 panels (was 17). Key changes: 3 tables converted to pie charts (Repeat Offenders, Detection Source, Assignment Group Load), Business Service Impact removed, MTTR Trend now split by P1 vs P2, new Alert vs Auto-Generated pie, new SLA monitoring strip (Breached / At Risk / Within tiles + full detail table). See `docs/offshore-email-v2.md` for the short version.

---

## 1. Import the Baseline

Before building any panel:

1. Go to **Kibana → Stack Management → Saved Objects → Import**
2. Upload `mim-dashboard-enhanced.ndjson`
3. On conflict: **Overwrite** existing objects
4. Verify import shows 3 objects saved (2 index patterns + 1 dashboard), 0 errors

---

## 2. KPI Status Reference

| KPI | Status | Dashboard Panel |
|-----|--------|----------------|
| # Active Major Incidents | ✅ Achieved | "Active P1/P2" metric tile, row 1 |
| MTTR (P1+P2) | ✅ Achieved (fixed) | "MTTR (Median, P1+P2)" metric tile, row 1 |
| Incidents by Service/Team | ✅ Achieved | "Incidents by CI / Support Team" datatable, row 3 |
| Recurrence Rate (% metric) | ✅ Achieved (new) | "Recurrence Rate (P1+P2)" metric tile + repeat-offender list, row 4 |
| Closure Rate % | ✅ Achieved (new) | "Closure Rate % (P1+P2)" metric tile, row 1 |
| MTTR Trend | ✅ Achieved (fixed) | "MTTR Trend (Median P1+P2, per day)" line chart, row 2 |
| Incident Volume Over Time | ✅ Achieved (new) | "Incident Volume Over Time (by Priority)" multi-series line, row 2 |
| Open Incident Ageing | ✅ Achieved (new) | "Open P1/P2 Incidents (Ageing)" table, row 4 |
| Detection Source | ✅ Achieved (new) | "Detection Source (All Incidents)" table, row 5 |
| Top Affected CIs | ✅ Achieved (new) | "Top Affected CIs" table, row 5 |
| Business Service Impact | ✅ Achieved (new) | "Business Service Impact (P1+P2)" table, row 5 |
| Assignment Group Load | ✅ Achieved (new) | "Assignment Group Load" table, row 6 |
| Long-Open Count (>24h) | ✅ Achieved (new) | "Long-Open P1/P2 (>24h)" metric tile, row 4 |
| MTTA | ❌ Not achievable | `acknowledged_at` field missing — see Section 4 |
| TTD (Declare Major) | ❌ Not achievable | `major_declared_at` field missing — see Section 4 |
| Time to Stabilize | ❌ Not achievable | `stabilized_at` field missing — see Section 4 |
| SLO Impact Rate | ❌ Not achievable | No SLO breach field — see Section 4 |
| RCA Timeliness | ❌ Not achievable | `rca_submitted_at` field missing — see Section 4 |

---

## 3. Building Each KPI — Step-by-Step Instructions

### 3.1 Active Major Incidents (metric tile)

Already built in the imported dashboard. If you need to rebuild it:

1. **Dashboards → Edit → Add panel → Lens**
2. Data view: `servicenow-incidents-*`
3. Visualization type: **Metric**
4. Drag `___records___` → **Primary metric** → Operation: **Count**
5. In the KQL filter bar: `priority: "1" OR priority: "2"`
6. Custom label: `Active P1/P2`
7. Colour palette: Custom — green (0–5), amber (5–15), red (15+)
8. Enable **Trendline**: yes

> **Prototype bug fixed**: the prototype used `ci.priority.keyword: ("1" OR "2")` — inconsistent with the `priority` field used in all other panels. Use `priority` only.

---

### 3.2 MTTR — Median Time to Resolve (ES|QL metric)

We compute hours **inside the query** and format as plain number, which sidesteps every Kibana Duration-format ambiguity (precision key name, unit inference, suffix visibility).

1. **Add panel → Lens**
2. Switch to **ES|QL** mode (top-left toggle)
3. Paste the query:
   ```esql
   FROM servicenow-incidents-*
   | WHERE opened_at IS NOT NULL AND resolved_at IS NOT NULL
       AND priority IN (1, 2)
   | EVAL mins = DATE_DIFF("minutes", opened_at, resolved_at)
   | WHERE mins > 0
   | STATS median_mins = MEDIAN(mins)
   | EVAL metric_val = ROUND(median_mins / 60.0, 1)
   ```
4. Visualization type: **Metric**
5. Map `metric_val` to the primary metric dimension
6. Format: **Number**, pattern `0.0`
7. Custom label: `MTTR (hrs)`

> **Why MEDIAN not AVG**: a single P1 incident with a 72-hour resolution time inflates the average by ~3×. Median is resistant to outliers and gives a more honest operational picture.

> **Prototype bug fixed**: the prototype used `AVG` and hardcoded 10 `ci.number` values in the WHERE clause. Both issues are corrected here.

---

### 3.3 Closure Rate % (ES|QL metric)

`COUNT(field)` in ES|QL counts non-null values, so `COUNT(resolved_at)` is exactly the number of incidents that have been closed. Multiplying by `100.0` (double) before dividing forces double-precision arithmetic.

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS total = COUNT(*), closed = COUNT(resolved_at)
| EVAL metric_val = ROUND(closed * 100.0 / total, 1)
```

Format: **Number**, pattern `0.0`. Put the `%` sign in the label: `Resolved / Opened (%)`.

---

### 3.4 MTTR Trend Line (ES|QL XY chart)

```esql
FROM servicenow-incidents-*
| WHERE opened_at IS NOT NULL AND resolved_at IS NOT NULL AND priority IN (1, 2)
| EVAL mins = DATE_DIFF("minutes", opened_at, resolved_at)
| WHERE mins > 0
| STATS metric_val = MEDIAN(mins) BY bucket = DATE_TRUNC(1 day, @timestamp)
| SORT bucket ASC
```

1. Visualization type: **Line**
2. X-axis: `bucket` (date), Y-axis: `metric_val` (median MTTR in minutes)

> **Prototype bug fixed**: the prototype's "MTTR Trend" panel showed `count(resolved_at)` over time — incident count, not MTTR. It was mislabelled.

---

### 3.5 Incident Volume Over Time (ES|QL XY chart)

```esql
FROM servicenow-incidents-*
| STATS metric_val = COUNT(*) BY bucket = DATE_TRUNC(1 day, @timestamp), priority
| SORT bucket ASC
```

X-axis: `bucket`, Y-axis: `metric_val`, split series by `priority`. Use stacked bar or multi-line.

---

### 3.6 Incidents by CI / Support Team (ES|QL datatable)

```esql
FROM servicenow-incidents-*
| STATS
    p1 = COUNT(CASE(priority == 1, number, null)),
    p2 = COUNT(CASE(priority == 2, number, null)),
    p3 = COUNT(CASE(priority == 3, number, null)),
    total_val = COUNT(*)
  BY ci_name = ci.name, team = ci.support_group.l2.name
| SORT total_val DESC
| LIMIT 50
```

Columns: CI Name | Support Team | P1 | P2 | P3 | Total

---

### 3.7 Recurrence Rate — real percentage + list of offenders

Two panels together: a **metric tile** with the actual recurrence rate as a %, and a **table** listing the repeat-offender CIs.

**Metric — recurrence rate as a percentage:**

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS n = COUNT(*) BY ci.name
| STATS repeats = COUNT(CASE(n > 1, 1, null)), total_cis = COUNT(*)
| EVAL metric_val = ROUND(repeats * 100.0 / total_cis, 1)
```

Format: **Number**, pattern `0.0`. Label: `% of CIs with >1 major incident`. Alias is `repeats` (not `repeat`) to avoid any collision with the `REPEAT()` ES|QL string function.

**Table — the repeat-offender list:**

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS inc_count = COUNT(*) BY ci_name = ci.name
| WHERE inc_count > 1
| SORT inc_count DESC
| LIMIT 25
```

Columns: CI Name | Incident Count. A CI appearing here needs an RCA and a permanent fix.

---

### 3.8 Open Incident Ageing (ES|QL datatable)

Shows unresolved P1/P2 incidents and how long they have been open:

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2) AND resolved_at IS NULL
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| KEEP number, ci.name, priority, age_hours, opened_at, assignment_group.name
| SORT age_hours DESC
| LIMIT 50
```

> Alert threshold guidance: P1 incidents open > 4 hours are breach-risk; P2 > 24 hours.

---

### 3.9 Detection Source (All Incidents)

Measured across **all incidents**, not just P1/P2. Alerting-coverage gaps show up more clearly in the lower-priority long tail.

```esql
FROM servicenow-incidents-*
| STATS metric_val = COUNT(*) BY contact_type
| SORT metric_val DESC
| LIMIT 10
```

If `contact_type = "Monitoring"` is dominant → good (self-detecting). If `contact_type = "Phone"` or `"Email"` dominates → incidents are user-reported, not proactively caught.

---

### 3.11 Business Service Impact (P1+P2)

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2) AND business_service.name IS NOT NULL
| STATS metric_val = COUNT(*) BY svc = business_service.name
| SORT metric_val DESC
| LIMIT 15
```

Ties MIM load to business capability — the conversation to have with service owners.

---

### 3.13 Long-Open P1/P2 Count (>24h)

Single-value tile showing the count of open P1/P2 incidents already older than 24 hours — the "how many alarms are red" number for the operations lead.

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2) AND resolved_at IS NULL
| EVAL age_hours = DATE_DIFF("hours", opened_at, NOW())
| WHERE age_hours > 24
| STATS metric_val = COUNT(*)
```

Format: **Number**, pattern `0,0`. Label: `Open incidents ageing past 24h`.

---

### 3.12 Assignment Group Load

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2) AND assignment_group.name IS NOT NULL
| STATS metric_val = COUNT(*) BY team = assignment_group.name
| SORT metric_val DESC
| LIMIT 15
```

Which teams handle the most major incidents. Concentration in one team = a resourcing conversation.

---

### 3.10 Action Closure Rate (Formulas approach)

The KPI spreadsheet links to an external Kibana dashboard for this. If `closed_at` is populated when resolved:

```esql
FROM servicenow-incidents-*
| STATS total = COUNT(*), resolved = COUNT(CASE(closed_at IS NOT NULL, number, null))
| EVAL metric_val = ROUND(resolved / total * 100.0, 1)
```

This overlaps with Closure Rate (Section 3.3). Treat as the same metric unless you have a separate "problem actions" table in ServiceNow.

---

## 4. KPIs That Cannot Be Achieved — What Is Required

These KPIs are blocked by missing fields in `servicenow-incidents-*`. Each entry describes exactly what needs to happen to enable it.

### 4.1 MTTA (Mean Time to Acknowledge)

**Calculation**: `AVG(acknowledged_at - opened_at)`

**Why blocked**: ServiceNow's Elastic integration does not export the "Acknowledge" state-change timestamp by default. The field does not exist in the index.

**How to fix**:
1. In ServiceNow, configure a Business Rule or Integration Hub flow that writes `u_acknowledged_at` whenever a user clicks "Acknowledge" on a major incident.
2. In the Elastic ServiceNow integration configuration, add `u_acknowledged_at` to the included fields list.
3. Once indexed, build: `DATE_DIFF("minutes", opened_at, u_acknowledged_at)`.

---

### 4.2 TTD — Time to Declare Major

**Calculation**: `AVG(major_declared_at - opened_at)`

**Why blocked**: ServiceNow's "Major Incident" flag is a boolean, not a timestamped event. The moment the flag is set is not exported.

**How to fix**:
1. Add a ServiceNow Business Rule that captures `sys_updated_on` when `major_incident_state` changes from `"Not a major incident"` to `"Candidate"` or `"Accepted"`.
2. Store this as `u_major_declared_at` on the incident record.
3. Export the field via the Elastic integration.

---

### 4.3 Time to Stabilize

**Calculation**: `AVG(stabilized_at - major_declared_at)`

**Why blocked**: No standard "system stabilized" event in ServiceNow. Would require a custom workflow step (e.g., a manual checkbox or update set that writes `u_stabilized_at`).

---

### 4.4 SLO Impact Rate

**Calculation**: `count(slo.breached = true) / count(major incidents)`

**Why blocked**: Elastic's SLO feature stores SLO breach records in a separate index (`slo-observability.sli-v3-*`), not in `servicenow-incidents-*`. A join between the two is not possible in Kibana Lens.

**How to fix**:
1. Enable Elastic SLO (requires Platinum licence or above).
2. Define an SLO for each Wave 1 application.
3. Use Elastic Transforms to denormalize: create a new index that attaches `slo.breached` to each incident based on time range overlap.
4. Build the panel against the transformed index.

---

### 4.5 RCA Timeliness

**Calculation**: `% of P1/P2 incidents where rca_submitted_at <= rca_due_at`

**Why blocked**: ServiceNow's Problem Management records (where RCAs live) are in a separate table (`problem`) and are not linked to the incidents index by default.

**How to fix**:
1. Configure the Elastic ServiceNow integration to also index the `problem` table.
2. Add `u_rca_submitted_at` and `u_rca_due_at` fields to the problem record.
3. Build a join via `problem.related_incident.number` in a Transform, or display RCA timeliness as a separate panel sourced from `servicenow-problem-*`.

---

## 5. Controls Configuration

The dashboard includes three filter controls. To verify they are wired correctly after import:

1. **Dashboards → Edit → Controls**
2. Three options-list controls should appear: **Priority**, **Assignment Group**, **CI Name**
3. Data view for each: `servicenow-incidents-*`
4. Fields: `priority`, `assignment_group.name`, `ci.name`
5. Chaining: HIERARCHICAL (selecting a priority narrows the team options)

---

## 6. Dashboard Settings

After import, set and save these once:

- **Time range**: Last 7 days
- **Auto-refresh**: Every 5 minutes
- **Store time with dashboard**: Save → tick "Store time with dashboard"

---

## 7. Recommended Additional KPIs (for future iterations)

| KPI | Why Add It | Build Approach |
|-----|-----------|---------------|
| **Escalation Rate** | Count of incidents that raised in priority (P3→P1/P2) | Requires priority-change history field; not currently available |
| **Incident Backlog Burn Rate** | Rate at which the open incident queue is shrinking | `COUNT(resolved_at IS NOT NULL) / COUNT(opened_at)` per day, plotted as line |
| **Repeat Assignee Involvement** | Identify individuals handling disproportionate MIM load | `COUNT(*) BY assignment_group.name, @timestamp` |
| **Short Description Clustering** | Detect recurring failure patterns by NLP keyword matching | Requires Elasticsearch ML / categorization aggregation |
| **Business Service Impact** | Which business services (`business_service.name`) are most affected | `COUNT(*) BY business_service.name` — straightforward with current fields |

---

*Prepared for offshore build team — September 2026.*
