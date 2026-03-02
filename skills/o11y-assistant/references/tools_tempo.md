# Tempo / Traces Tools & Cost Rules

> Loaded by SKILL.md at Step 4 when querying trace backends.
> Covers: Tool parameters, query type validation, backend-specific notes, TraceQL cost rules.

---

## Tool Reference

**Core Requirement:** ALL Tempo tools require `datasourceUid` parameter (string, camelCase, required).

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `tempo_get-attribute-names` | `datasourceUid`, `scope` (optional: span/resource/event/link/instrumentation) | Discover available attributes. Omit scope to see all. | Scope reduces results to single type |
| `tempo_get-attribute-values` | `datasourceUid`, `name`, `filter-query` (optional) | Discover values for scoped attribute | 🔴 **filter-query:** Exactly ONE spanset, && only (no \|\|). Format: `{ cond && cond }`. |
| `tempo_traceql-search` | `datasourceUid`, `query`, `start`, `end` (optional), `limit` | Execute TraceQL **search query** (finds traces). | ❌ Do NOT send metrics queries (aggregations). Use metrics-instant/range instead. |
| `tempo_get-trace` | `datasourceUid`, `trace_id` | Retrieve full trace by ID | — |
| `tempo_traceql-metrics-instant` | `datasourceUid`, `query`, `start`, `end` (optional) | Execute TraceQL **metrics query**. Single instant value. For: count(), rate(), quantile(), avg(). | ❌ Do NOT send search queries. |
| `tempo_traceql-metrics-range` | `datasourceUid`, `query`, `start`, `end` (optional) | Execute TraceQL **metrics query** with time-series output. Trend analysis. | Same as metrics-instant. |
| `tempo_docs-traceql` | `datasourceUid`, `name` (basic/aggregates/structural/metrics) | Reference TraceQL syntax and operators | — |

---

## Query Type Validation

| Query Type | Keywords | Use Tool | ❌ Do NOT Use |
|------------|----------|----------|--------------|
| **Search** | `{ }` span/resource filter, no aggregation | `tempo_traceql-search` | `tempo_traceql-metrics-*` |
| **Metrics** | `count()`, `rate()`, `quantile()`, `avg()`, `sum()` | `tempo_traceql-metrics-instant` or `-range` | `tempo_traceql-search` |

**Error messages guide you:** "TraceQL metrics query received on search tool" → wrong tool for query type.

---

## Backend-Specific Query Notes

- **Identify query type first:**
  - Find traces: `tempo_traceql-search` with `{ resource.service.name="<svc>" && <condition> }`
  - Aggregate: `tempo_traceql-metrics-instant` or `-range` with `count()`, `rate()`, `quantile()`
- **Common filters:** Latency: `duration > 500ms` | Errors: `status = error` | HTTP: `span.http.status_code = 500`
- **Large datasets:** Use metrics aggregations instead of search
- **Deep dive:** `tempo_get-trace(trace_id=...)` for single-trace analysis
- **Attribute discovery:** `tempo_get-attribute-names(scope="span")` to discover available span attributes

---

## TraceQL Cost Rules (Appendix C)

### Foundational Principle

Tempo stores data in Parquet columnar format. Cost = columns read × I/O. Only &&-only queries enable predicate pushdown into Parquet layer.

### Mandatory Construction Rules

1. **Lead with trace-level intrinsics** (trace:rootService, trace:duration, trace:rootName)
2. **Always scope attributes** (span., resource. — never unscoped .attr)
3. **&& within single { } selector** for same-span conditions (enables pushdown)
4. **Exact equality over regex** (= over =~)
5. **resource/trace columns before span attributes** (smaller columns first)
6. **Use dedicated OTel columns** over generic attributes:
   - span.http.method / span.http.status_code / span.http.url / span.db.system
   - resource.service.name / resource.cloud.region / resource.deployment.environment
   - trace:duration / trace:rootService / trace:rootName
   - status / duration / name / kind (span intrinsics — indexed)
7. **Minimize column count** — drop redundant conditions
8. **Avoid || when && semantically equivalent** (|| prevents pushdown)
9. **Avoid structural operators unless required** (>> , >, ~ most expensive)
10. **Use sampling for large metrics queries:** `| quantile_over_time(duration, 0.9) with(sample=true)`
11. **Use span:childCount and nil checks (Tempo 2.10+)** for leaf span detection

### Warning Triggers

```
🔴 .attr (unscoped)                  UNSCOPED — forces span+resource lookup
🔴 =~ ".*text.*" (greedy)           GREEDY REGEX — replace with exact =
🔴 { A } && { B } same-span intent  SPLIT SELECTOR — loses pushdown; merge
🔴 >> operator                       ANCESTOR TRAVERSAL — most expensive
🟡 =~ or !~ on any attribute         REGEX MATCH — verify cannot be =
🟡 || condition                      OR — prevents predicate pushdown
🟡 quantile_over_time no sample=true NON-SAMPLED QUANTILE — add with(sample=true)
```

### TraceQL Query Checklist

- [ ] Leads with trace: intrinsic or resource.service.name
- [ ] All attributes explicitly scoped (span., resource.)
- [ ] Same-span && conditions in single { } selector
- [ ] No regex where exact match possible
- [ ] No || unless OR logic genuinely required
- [ ] No >> / ~ unless cross-span relationship explicitly required
- [ ] `quantile_over_time` uses `with(sample=true)` for large datasets
- [ ] Column count minimized; no redundant conditions
- [ ] Dedicated OTel columns preferred
