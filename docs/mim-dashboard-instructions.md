# MIM Dashboard — Consultant Build Instructions

Audience: offshore Kibana engineers building the enhanced MIM dashboard.  
Source index: `servicenow-incidents-*`  
Kibana version: 8.x (tested at 8.8+)

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
| Recurrence Rate | ✅ Achieved (new) | "Recurrence — CIs with Multiple P1/P2 Incidents" table, row 4 |
| Closure Rate % | ✅ Achieved (new) | "Closure Rate %" metric tile, row 1 |
| MTTR Trend | ✅ Achieved (fixed) | "MTTR Trend (Median P1+P2, per day)" line chart, row 2 |
| Incident Volume Over Time | ✅ Achieved (new) | "Incident Volume Over Time" line chart, row 2 |
| Open Incident Ageing | ✅ Achieved (new) | "Open P1/P2 Incidents (Ageing)" table, row 4 |
| Contact Type Breakdown | ✅ Achieved (new) | "Contact Type (P1/P2)" table, row 5 |
| Top Affected CIs | ✅ Achieved (new) | "Top Affected CIs" table, row 5 |
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

1. **Add panel → Lens**
2. Switch to **ES|QL** mode (top-left toggle)
3. Paste the query:
   ```esql
   FROM servicenow-incidents-*
   | WHERE opened_at IS NOT NULL AND resolved_at IS NOT NULL
       AND priority IN (1, 2)
   | EVAL mins = DATE_DIFF("minutes", opened_at, resolved_at)
   | WHERE mins > 0
   | STATS metric_val = MEDIAN(mins)
   ```
4. Visualization type: **Metric**
5. Map `metric_val` to the primary metric dimension
6. Format: **Duration** → Input format: Minutes → Output: Hours (1 decimal)
7. Custom label: `MTTR (hours)`

> **Why MEDIAN not AVG**: a single P1 incident with a 72-hour resolution time inflates the average by ~3×. Median is resistant to outliers and gives a more honest operational picture.

> **Prototype bug fixed**: the prototype used `AVG` and hardcoded 10 `ci.number` values in the WHERE clause. Both issues are corrected here.

---

### 3.3 Closure Rate % (ES|QL metric)

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS total = COUNT(*),
        closed = COUNT_DISTINCT(CASE(resolved_at IS NOT NULL, number, null))
| EVAL metric_val = ROUND(closed / total * 100.0, 1)
```

Format: **Percent** (1 decimal). Label: `Closed / Total (%)`.

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

### 3.7 Recurrence Rate — CIs with Multiple P1/P2 Incidents

This KPI answers: "Which CIs keep breaking?" — i.e., repeat major incidents on the same application within the dashboard's time window.

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS inc_count = COUNT(*) BY ci_name = ci.name
| WHERE inc_count > 1
| SORT inc_count DESC
| LIMIT 25
```

Columns: CI Name | Incident Count

> This is achievable — the KPI spreadsheet marked it "Can be achieved" and this implements it. A CI appearing here needs an RCA and a permanent fix.

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

### 3.9 Contact Type Breakdown

```esql
FROM servicenow-incidents-*
| WHERE priority IN (1, 2)
| STATS metric_val = COUNT(*) BY contact_type
| SORT metric_val DESC
| LIMIT 10
```

If `contact_type = "Monitoring"` is dominant → good (self-detecting). If `contact_type = "Phone"` or `"Email"` dominates → incidents are user-reported, not proactively caught.

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
