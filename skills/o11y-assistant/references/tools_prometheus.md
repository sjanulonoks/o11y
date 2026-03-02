# Prometheus / Mimir Tools & Cost Rules

> Loaded by SKILL.md at Step 4 when querying metrics backends.
> Covers: Tool parameters, backend-specific query notes, PromQL cost rules.

---

## Tool Reference

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `list_prometheus_metric_names` | `datasourceUid`, `regex` (optional) | Discover available metrics. Supports pagination. | ⚠️ Default limit truncates results. Re-query with higher limit if incomplete. |
| `list_prometheus_metric_metadata` | `datasourceUid`, `metric` (optional) | Confirm metric type (counter/gauge/histogram) | Experimental endpoint |
| `list_prometheus_label_names` | `datasourceUid`, `matches` (optional), time range (optional) | Discover label keys. `matches`: `[{filters: [{name, type, value}]}]` where type ∈ {=, !=, =~, !~}. | Regex in `=~` must be anchored (no leading `.*`) |
| `list_prometheus_label_values` | `datasourceUid`, `labelName`, `matches` (optional), time range | Discover label values for label-metric combination | Same `matches` rules as label_names |
| `query_prometheus` | `datasourceUid`, `expr`, `startTime`, `queryType` ("instant"/"range"), `endTime` (range only), `stepSeconds` (range only) | Execute PromQL. | Empty matrix/vector = metric doesn't exist OR no data in range. NaN = query matches but data is NaN. |
| `query_prometheus_histogram` | `datasourceUid`, `metric` (base name, no _bucket), `percentile` (0-100), `labels` (string), `rateInterval` | Generate histogram_quantile PromQL. Returns time-series at percentile. | Labels format: `"key=\"value\",key2=\"value2\""` (not map) |

---

## Backend-Specific Query Notes

- **Existence check:** Start with `up{job="<value>"}` to confirm service exists
- **Step parameter (range queries):** expected_samples ≈ (endTime − startTime) / step. Start with 60s. Step ≥ scrape interval (15s–60s).
- **Histogram queries:** Use `query_prometheus_histogram(metric="base_name", percentile=95, labels="job=\"api\"", rateInterval="5m")`

---

## PromQL Cost Rules (Appendix A)

### Mandatory Construction Rules

1. **Prefer recording rules** for frequently-used aggregations
2. **Most selective exact-match labels first** in every selector
3. **Prefer exact match over regex** (= over =~; anchored alternation =~"a|b" acceptable)
4. **Always anchor regex** — never lead with `.*word` (suffix matching kills index)
   - ✅ `=~"^prometheus"` (prefix), `=~"error$"` (suffix), `=~"^api|^gateway$"` (anchored alternation)
   - ❌ `=~".*prometheus.*"` (leading wildcard), `=~"(?i)error"` (case-insensitive)
5. **Minimize time range; maximize step interval** (step ≥ scrape interval always)
6. **Avoid high-cardinality labels** (user_id, request_id, trace_id) in `by` aggregations
7. **Avoid subqueries** — use recording rules instead
8. **Use `without` instead of `by`** when dropping only 1–2 labels
9. **Prefer native histograms** over classic (Mimir 3.0+)
10. **Structure for MQE sharding** — aggregate before joining
11. **Keep sub-selectors identical** for MQE common-subexpression dedup

### Warning Triggers

```
🔴 {label=~".*word.*"}              UNANCHORED WILDCARD — disables index
🔴 sum by (user_id|request_id)()   HIGH-CARDINALITY AGG — no reduction
🔴 nested subquery                  NESTED SUBQUERY — multiplied cost
🔴 range > 24h + step < scrape     OVER-RESOLUTION — wasted evaluations
🟡 bare metric name, no labels      NO LABEL FILTER — add job= minimum
🟡 subquery                         SUBQUERY — use recording rule
```

### PromQL Query Checklist

- [ ] Most selective exact-match labels first
- [ ] No unanchored wildcard regex (`.*word.*`)
- [ ] No bare metric name without at least `job=` label
- [ ] No high-cardinality `by` (user_id / request_id / trace_id)
- [ ] No subquery without recording rule alternative noted
- [ ] Step ≥ scrape interval
- [ ] Time range as narrow as needed
- [ ] Identical sub-selectors for MQE dedup
