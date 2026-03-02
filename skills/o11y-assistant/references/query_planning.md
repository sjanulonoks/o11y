# Query Planning Framework

**MANDATORY: Complete this framework before ANY analytical query to ANY backend.**

> This file is loaded by the main SKILL.md workflow at Step 4.
> Return to SKILL.md after completing query planning.

---

## The Iron Rule

```
🔴 FORBIDDEN: Deriving answers through calculation when direct query exists

❌ Query rate → multiply by time → present estimate
✅ Query directly for what user asked → return exact value

Why: Calculations introduce error. Backends store raw data. Query it directly.
```

---

## Universal Intent → Function Matrix

Classify user intent, then map to backend-specific direct query function:

| Intent | User Keywords | Prometheus | Loki | Tempo |
|--------|---------------|------------|------|-------|
| **COUNT/TOTAL** | "how many", "total", "count" | `sum(increase(metric[15m]))` or `count_over_time()` | `count_over_time({selector}[15m])` | `count_over_time()` (Instant, step=window) |
| **RATE** | "per second", "TPS", "QPS" | `rate(metric[5m])` | `rate({selector}[5m])` | `rate()` |
| **DISTRIBUTION** | "p50", "p95", "p99", "percentile" | `histogram_quantile(0.95, metric_bucket)` | N/A | `quantile_over_time(duration, 0.95)` |
| **CENTRAL TENDENCY** | "average", "mean" | `avg_over_time(metric[15m])` | N/A | `avg_over_time(duration)` |
| **RANGE** | "min", "max", "highest", "lowest" | `max_over_time(metric[15m])` | N/A | `max_over_time(duration)` |
| **EXISTENCE** | "is X up", "does X exist" | `up{job="X"}` | `{selector}` limit=1 | `{ service="X" }` limit=1 |
| **COMPARISON** | "difference", "delta", "vs" | Two instant queries, compare | Two instant queries | Two instant queries |
| **TREND** | "over time", "per minute", "pattern" | Range query | Range query | Range query |

---

## Instant vs Range Query Decision

| User Says | Query Type | Returns | Example |
|-----------|------------|---------|---------|
| "What is...", "How many...", "Total..." | **Instant** | Single value | "What is total request count?" |
| "Show trend...", "Over time...", "Per minute..." | **Range** | Time-series | "Show request rate per minute" |
| "Compare X vs Y" | **Instant** × 2 | Two values | "Requests in EU vs US" |

---

## Query Plan Template

**Output this BEFORE every analytical query:**

```
## QUERY PLAN
Backend: [Prometheus/Loki/Tempo/other]
User intent: [COUNT/RATE/DISTRIBUTION/etc]
User's exact words: "[quote]"
Data structure needed: [single value / time-series / comparison]
Query type: [Instant / Range]
Function/aggregation: [specific function name]
Rationale: User asked for [X], directly provided by [function]

Anti-pattern check: ❌ NOT using [wrong approach] because [reason]
```

### Query Planning Checklist

- [ ] User intent classified (COUNT/RATE/DISTRIBUTION/etc)
- [ ] Direct query function identified (not derived calculation)
- [ ] Query type matches intent (Instant for values, Range for trends)
- [ ] Anti-pattern explicitly checked
- [ ] Query plan documented above

---

## Cardinality Validation Protocol

**APPLIES TO:** Any query where user asks "how many" or "total count" of entities.

**WHY:** Cardinality confusion is the most common root cause of incorrect counts:
- Prometheus: 1 node might = 3 time series
- Loki: 1 error event might = 2-5 log lines
- Tempo: 1 request might = 5-15 spans

### Gate 1: Semantic Entity Definition

```
Entity:              [what you're measuring: node, service, pod, request, etc.]
Backend raw unit:    [what backend measures: metric instance, log line, span]
Cardinality model:   [1:N ratio and mechanism]
Assumption:          [why you believe ratio is N]
```

### Gate 2: Cardinality Discovery (Sample Before Total)

**MANDATORY: Never report totals without sampling cardinality first.**

1. Execute discovery query to see sample of raw data (not aggregated)
2. Inspect labels/structure to understand one raw unit
3. Count distinct raw units for ONE entity label value
4. Document observed cardinality ratio

```
Query: kube_node_info (raw metric instances)
Result: 14 instances with unique node labels
Cardinality observed: 1 metric per node ✓
Ratio: 1:1
```

**Red flag:** Reporting total without sampling, or assuming 1:1 without verification.

### Gate 3: Measurement Translation

```
Gate 3 Translation:
Raw count:              [from Gate 2]
Observed ratio:         [from Gate 2]
Calculation:            [raw] ÷ [ratio] = [entity count]
Final answer:           [number] [entity type]
Verification:           Does [entity count] match expectations? Yes/No + explanation
```

### Gate 4: Cross-Backend Validation

**MANDATORY: If multiple backends available, reconcile answers BEFORE reporting.**

1. For each backend: attempt to answer same COUNT question
2. Apply Gates 1–3 for each backend
3. Compare final entity counts
4. If different: **STOP and investigate WHY** before reporting

**"Cannot apply" is NOT Skip:** Document attempt and why it failed.

### Cardinality Validation Checklist

- [ ] Gate 1 WRITTEN: Entity definition + raw unit + cardinality model
- [ ] Gate 2 EXECUTED: Sample of raw data retrieved
- [ ] Gate 2 DOCUMENTED: Cardinality ratio observed from actual data
- [ ] Gate 3 WRITTEN: Translation calculation shown
- [ ] Gate 3 VERIFIED: Final answer matches translation exactly
- [ ] Gate 4 ATTEMPTED: Same question attempted in each available backend
- [ ] Gate 4 RECONCILED: Conflicting results investigated (or consistent count documented)
