# Analytical Backend Tool Reference

> Loaded at Step 4 entry. Read ONLY the section for the backend you are about to query.

---

## Prometheus / Mimir

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `list_prometheus_metric_names` | `datasourceUid`, `regex` (optional) | Discover available metrics |
| `list_prometheus_metric_metadata` | `datasourceUid`, `metric` (optional) | Confirm metric type (counter/gauge/histogram) |
| `list_prometheus_label_names` | `datasourceUid`, `matches` (optional) | Discover label keys |
| `list_prometheus_label_values` | `datasourceUid`, `labelName`, `matches` (optional) | Discover label values |
| `query_prometheus` | `datasourceUid`, `expr`, `startTime`, `queryType` ("instant"/"range"), `endTime`, `stepSeconds` | Execute PromQL. Empty result = no data, not always "no problem". ⚠️ **USE ONLY** for non-distribution queries. For percentiles, use `query_prometheus_histogram`. |
| `query_prometheus_histogram` | `datasourceUid`, `metric` (base, no _bucket), `percentile` (0-100), `labels`, `rateInterval` | Generate histogram_quantile. Labels: `"key=\"value\""` ⚠️ **USE ONLY** for histogram metrics. |

**Prometheus:** Start with `up{<service_label>="<svc>"}` (use label from Session State service mapping) to confirm service exists. Empty? → convention-first discovery: try `job`, `service`, `app` via `list_prometheus_label_values`. Step ≥ scrape interval (start with 60s). Use `query_prometheus_histogram` for percentiles. **Baseline:** For key metrics, compare across two horizons:
- **Trend** (is it worsening?): `offset 5m` → `offset 15m` → `offset 45m` — shows direction within the incident
- **Seasonal baseline** (is this abnormal?): `offset 7d` / `offset 14d` — same day-of-week comparison. Avoid `offset 1d` (weekend/weekday seasonality misleads).

---

## Loki

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `list_loki_label_names` | `datasourceUid` | Discover log stream label keys |
| `list_loki_label_values` | `datasourceUid`, `labelName` | Discover values. 🟡 Filter higher-level label first. |
| `query_loki_stats` | `datasourceUid`, `logql` (selector only) | **MANDATORY before broad log pull.** Returns streams, chunks, entries, bytes. |
| `query_loki_logs` | `datasourceUid`, `logql`, `limit` (default 10, max 100), `direction`, `queryType` | Execute LogQL. Start limit=10. 🔴 Log timestamps = nanosecond strings. ⚠️ **USE ONLY** when LogQL control (metrics, filters) is needed. For quick keyword scans, use `search_logs`. |
| `search_logs` | `DatasourceUID`, `Pattern`, `Start`, `End`, `Limit` | Quick text/regex search. Auto-generates queries. ⚠️ **USE ONLY** for quick text searches. |

**Loki:** `query_loki_stats` MANDATORY before broad pulls (>1M entries → narrow first). Start limit=10, expand cautiously. Direction: "backward" (newest first) for recent events.

---

## Tempo / Traces

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `tempo_docs-traceql` | `datasourceUid`, `name` ("basic"/"aggregates"/"structural"/"metrics") | Self-teach TraceQL syntax — call before constructing unfamiliar query type |
| `tempo_get-attribute-names` | `datasourceUid`, `scope` (optional: "span"/"resource"/"event") | Discover available trace attributes |
| `tempo_get-attribute-values` | `datasourceUid`, `name` | Discover values. 🔴 `filter-query`: single `{ }`, `&&` only (no `\|\|`). |
| `tempo_traceql-search` | `datasourceUid`, `query`, `start`, `end`, `limit` | **Search queries** (find traces). ❌ SEARCH FILTERS ONLY (no aggregations). |
| `tempo_traceql-metrics-instant` | `datasourceUid`, `query`, `start`, `end` | **Metrics queries** (count, rate, quantile, avg, compare). Single value. |
| `tempo_traceql-metrics-range` | `datasourceUid`, `query`, `start`, `end` | **Metrics queries** with time-series output. |
| `tempo_get-trace` | `datasourceUid`, `trace_id` | Retrieve full trace by ID |

**Tempo query type validation:**

| Type | Keywords | Use Tool |
|------|----------|----------|
| **Search** | `{ }` filter, no aggregation | `tempo_traceql-search` |
| **Metrics** | `count()`, `rate()`, `quantile()`, `avg_over_time()` | `tempo_traceql-metrics-instant` or `-range` |
| **Compare** | `\| compare({condition})` — diff errors vs success, v1 vs v2 | `tempo_traceql-metrics-instant` or `-search` |

**Structural operators** (use `with(trace_sample=0.1)` when any are present):

| Op | Returns | Typical use |
|----|---------|------------|
| `>>` | Descendants | Find all downstream calls |
| `<<` | Ancestors | Find all upstream callers |
| `>` / `<` | Direct child / parent | Immediate dependency only |
| `~` | Siblings | N+1 / parallel operation detection |
| `!<` | No parent error | Find cascade root cause (origin error) |
| `&>>` | Full downstream path including self | Complete error cascade |

---

**Sampling — append at END of pipeline (MANDATORY for large datasets):**

| When | Method | Syntax |
|------|--------|--------|
| Default: unknown volume / window >15 min | `with(sample=true)` | `{ } \| rate() with(sample=true)` |
| Any structural operator present (`>>`, `<<`, `~`) | `with(trace_sample=0.1)` | `{ } >> { } \| rate() with(trace_sample=0.1)` |
| Exact control, data characteristics known | `with(span_sample=0.1)` | `{ } \| rate() with(span_sample=0.1)` |

Always for 7-day+ windows. Never mix methods in one query.

```
🔴 { selector } with(sample=true) | rate()   ← WRONG (sampling before pipeline end)
✅ { selector } | rate() with(sample=true)   ← CORRECT (at pipeline END)
```

---

**Traces as high-cardinality discovery** — use before falling back to multiple metric queries.

Traces carry endpoint, version, status code, downstream service, DB statement — attributes Prometheus never has. One sampling query often replaces 3–5 Prometheus queries + manual correlation.

| Diagnostic question | Query pattern |
|---------------------|--------------|
| Which endpoint is slow/erroring? | `{ svc } \| rate() by (span.http.target, span:status) with(sample=true)` |
| Which version regressed? | `{ svc } \| compare({resource.service.version="v2"})` |
| Which downstream causes errors? | `{ svc && span:status=error } >> { } \| rate() by (resource.service.name) with(trace_sample=0.1)` |
| What differs between errors and successes? | `{ svc } \| compare({span:status=error})` |
| N+1 / parallel DB calls? | `{ span.db.system!=nil } ~ { span.db.system!=nil }` |
| Cascade root cause (origin error)? | `{ span:status=error } !< { span:status=error }` |

Use `\| topk(N)` to limit high-cardinality groupings. Use `trace:rootService="X"` (indexed) instead of `resource.service.name="X" && span:kind=server` (slower).

---

## Utility — Deeplinks

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `generate_deeplink` | `resourceType` (dashboard/panel/explore), type-specific UIDs | Create shareable Grafana links. Always include time range. |

---

## Time Format Reference

| Format | Example | Valid |
|--------|---------|-------|
| **Relative (preferred)** | `"now-1h"`, `"now-30m"`, `"now-1d"` | ✅ |
| **Absolute (RFC3339 UTC)** | `"2024-01-15T10:00:00Z"` | ✅ (Z required) |
| Fractional / natural language / unix | — | ❌ |
