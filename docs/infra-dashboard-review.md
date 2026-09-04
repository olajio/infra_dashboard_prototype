# Infra Dashboard Prototype — Review

Review of the **Infra Dashboard Prototype** as it exists in `export.ndjson`
(1 dashboard, 19 panels, sourced from `metrics-*` and `servicenow-incidents-*`).

> A styled, shareable version of this review — including a visual mock-up of the
> target-state dashboard — is in `docs/infra-dashboard-review.html`
> (published artifact: https://claude.ai/code/artifact/019c9de5-6c97-42c0-9c17-a86b33085822).

## Executive summary

The prototype is a good skeleton pointed at the right data: it pulls from the two
sources that matter (system metrics + ServiceNow incidents) and the CPU/memory heat
tables are genuinely useful. But it isn't yet safe to run a NOC on:

- the **up / down / availability** numbers depend on settings the dashboard doesn't pin down,
- the **incident** panel isn't proven to be about infrastructure at all,
- **Oracle & MySQL** are empty placeholder sections with duplicate links,
- and there are **no filters, no history, and no auto-refresh**.

**9 corrections** to what's already there (3 are the issues you flagged) and
**11 enhancements** that aren't there yet. All **5 of your observations are valid**;
two need one fact confirmed from the live environment.

## Your five notes — assessed

| # | Your note | Verdict |
|---|-----------|---------|
| 1 | Server Down `now-5m` → `now-15m` | **Agree.** Up = `@timestamp >= now-5m`, Down = total − hosts-seen-in-5m. A reboot/agent-restart/ingest gap easily crosses 5 min. Move to `now-15m`; also unify quoted `"now-5m"` (Up) vs unquoted `now-5m` (Down/Avail). |
| 2 | "Fleet" → "Server Availability" | **Agree.** Tile says *Fleet Availability %* while the section header says *Servers Availability* — contradictory. Rename the tile. |
| 3 | Are P1/P2/P3 actually infra incidents? | **Agree — needs a data answer.** Panels filter on `priority_name` only, no host/CI link; the incidents index carries no `host.*` field (why you couldn't relate records in Discover). Confirm a CI/category field (`cmdb_ci`, `ci.name`, `u_host`, `category`); then scope to infra, or relabel/relocate honestly. |
| 4 | Oracle "Dashboard 1/2/3" all same URL | **Agree.** All three resolve to `…/app/r/s/tf0R3`. Wire each to its real destination + functional name, or collapse to one. |
| 5 | MySQL same duplicate links | **Agree.** Identical problem and fix as note 4. Both Oracle & MySQL also contain *only* links — no metrics of their own (see C6). |

## Corrections — things already on the dashboard

| ID | Correction | Now → target | Severity |
|----|------------|--------------|----------|
| C1 | Widen down-detection | `now-5m` → `now-15m`, consistent syntax *(note 1)* | High |
| C2 | Rename tile | "Fleet Availability %" → "Server Availability %" *(note 2)* | Low |
| C3 | Fix availability baseline | Denominator = `unique_count(host.name)` over the current picker range, so "monitored"/"down" drift with the range → count against a stable baseline (CMDB `ci.is_monitored` / `ci.operational_status`) | High |
| C4 | Prove incidents are infra | `priority_name`-only filter, no host/CI link → add CI/category filter, or relabel & relocate *(note 3)* | High |
| C5 | Fix detailed-dashboard links | Oracle & MySQL "Dashboard 1/2/3" → same `tf0R3` URL → real destinations + names *(notes 4 & 5)* | Medium |
| C6 | Populate or drop Oracle & MySQL | Both sections have links only, zero metrics → add DB-tier panels or hide until ready | Medium |
| C7 | "Linux" only matches RedHat | Filter `host.os.family == "redhat"` excludes Ubuntu/Debian/CentOS/SUSE/Amazon → broaden, or rename "RHEL Servers" | Medium |
| C8 | Clean up the tables | 4 tables still titled `Bar vertical stacked` with duplicate column defs → real titles + tidy columns | Low |
| C9 | Save default range + auto-refresh | `timeRestore:false`, `refreshInterval:null` → default Last 1h, refresh 1m, "Store time with dashboard" | High |

## Enhancements — not on the dashboard yet

| ID | Enhancement | What it adds | Effort |
|----|-------------|--------------|--------|
| E1 | Filter / control bar | Controls for `relay.dc`, `host.os.family`, `ci.operational_status`, host search | Low |
| E2 | Trends, not point-in-time | Availability-over-time line + CPU/mem trend | Med |
| E3 | "Down hosts" list | The actual list (host, last-seen, DC, OS, owner), not just a count | Low |
| E4 | Disk & network | `system.filesystem.used.pct`, network throughput/errors — disk-full is invisible today | Med |
| E5 | SLO & thresholds/alerts | Availability SLO + error budget, reference lines, Elastic alerting rules | Med |
| E6 | Incident context | Over-time by priority, open vs resolved, MTTR/ageing, top CIs | Med |
| E7 | Service & process health | `windows.service.*`, `process.*`, `system.process.state` already ingested | Low |
| E8 | CMDB enrichment | Group by `ci.name` / `ci.geo.name`; ownership; anchor the availability baseline | Med |
| E9 | Monitor the monitor | `ingest_lag_in_sec` + `error.message` freshness panel | Low |
| E10 | One fleet model + coverage | Define expected fleet from CMDB; show coverage gaps | Med |
| E11 | Legends, units & "as of" | Heat-threshold legend, explicit units, "data as of" indicator | Low |

## How to make the changes

Everything is in Kibana → Dashboards → **Infra Dashboard Prototype** → **Edit**.

### Phase 1 — Corrections (make today's numbers trustworthy)

**C1 — Widen the window (Servers Up / Down / Availability):**
```
# Servers Up — filter on the unique_count column
@timestamp >= now-15m

# Servers Down — formula
unique_count(host.name) - unique_count(host.name, kql='@timestamp >= now-15m')

# Server Availability % — formula
unique_count(host.name, kql='@timestamp >= now-15m') / unique_count(host.name)
```

**C2 —** Rename the metric's custom label `Fleet Availability %` → `Server Availability %`.

**C3 — Anchor the baseline (recommended):**
```
# denominator using CMDB attributes present on metrics-*
unique_count(host.name, kql='ci.is_monitored: true and ci.operational_status: "Operational"')
```
If CMDB isn't reliable yet, at minimum pin the time range (C9) so the % is defined.

**C4 — Prove / relabel incidents.** In Discover on `servicenow-incidents-*`, look for a
CI/category field (`cmdb_ci`, `ci.name`, `u_host`, `category`). Then:
```
# if a CI field exists — add to each panel's KQL filter
priority_name : "1 - Critical" and cmdb_ci : *

# if no host/CI link exists — rename section "ServiceNow Incidents (all)" and relocate
```

**C5 — Real links (edit Oracle & MySQL markdown panels):**
```
### Oracle — Detailed Dashboards ###
[Tablespace & Sessions](https://kibana-prod.gcp.cna.com/app/r/s/<id-1>)
[Wait Events & Locks](https://kibana-prod.gcp.cna.com/app/r/s/<id-2>)
[Backup & Data Guard](https://kibana-prod.gcp.cna.com/app/r/s/<id-3>)
```

**C6 —** Add DB-tier panels to Oracle/MySQL, or hide the sections until data exists.

**C7 — Make "Linux" mean Linux:**
```
FROM .ds-metrics-system.cpu-default*
  | WHERE host.os.family != "windows" AND system.cpu.total.norm.pct IS NOT NULL
  | STATS cpu_pct = AVG(system.cpu.total.norm.pct) BY host.name, host.os.family
  | SORT cpu_pct DESC
```

**C8 —** Rename the four tables (e.g. "Windows — CPU % by host") and drop duplicate columns.

**C9 —** Time picker → Last 1h, auto-refresh 1m, **Save** with **"Store time with dashboard"**.

### Phase 2 — Enhancements (turn it into an observability tool)

**E1 —** Add Controls: options lists for `relay.dc`, `host.os.family`, `ci.operational_status`
+ a `host.name` search (chaining on).

**E3 — Servers not reporting (new ES|QL table):**
```
FROM .ds-metrics-system.cpu-default*
  | STATS last_seen = MAX(@timestamp) BY host.name, host.os.family, relay.dc
  | WHERE last_seen < NOW() - 15 minutes
  | SORT last_seen ASC
```

**E4 — Disk usage (new ES|QL table):**
```
FROM .ds-metrics-system.filesystem-default*
  | WHERE system.filesystem.used.pct IS NOT NULL
  | STATS disk_pct = MAX(system.filesystem.used.pct) BY host.name, system.filesystem.mount_point
  | WHERE disk_pct > 0.8
  | SORT disk_pct DESC
```

**E7 — Stopped services (new ES|QL table):**
```
FROM .ds-metrics-windows.service-default*
  | WHERE windows.service.state != "Running" AND windows.service.start_type == "Automatic"
  | KEEP host.name, windows.service.display_name, windows.service.state
  | SORT host.name
```

**E2 —** Lens line: unique_count(`host.name`) over a date histogram; plus CPU/mem trend.
**E9 —** Lens metric: median `ingest_lag_in_sec`, agents reporting vs expected, `error.message` count.
**E5 / E6 —** Elastic SLO (99.9%) + error budget, reference lines, alerting rules; incident
over-time by priority, open vs resolved, MTTR/ageing, top CIs.
**E8 / E10 / E11 —** CMDB grouping & ownership, fleet-coverage view, threshold legend + units + "as of".

### Suggested order of play

1. Ship **Phase 1** as one low-risk change (closes your five notes, makes numbers honest).
2. **E1 filters + E3 down-list + E9 pipeline health** (fast, high value).
3. **E4 disk + E2 trends.**
4. **E5/E6 SLO & incident context** once C4's data question is answered.
