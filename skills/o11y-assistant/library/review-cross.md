# --review-cross Protocol

**Usage:** `--review-cross <svc-A> [<svc-B> <svc-C> ...]`

Standalone operation. No active investigation required. Reads grade entries from
multiple services and produces a cross-service pattern analysis.

## Step 1: Load

For each service specified:
- Read `resolutions/<svc>.md`. Extract all YAML `# grade-entry` blocks (Phase 4 format, v0.57+).
- If a service file doesn't exist → note it as missing; continue with remaining services.
- If fewer than 2 services have YAML entries → emit: `[REVIEW-CROSS: insufficient YAML history — run --grade investigations first for all services]` and stop.

## Step 2: Cross-Service Analysis

### 2a — Shared Root Causes
Group root causes across ALL services by semantic similarity.
For each cluster:
- List which services are affected and how many times each
- Note if same Trigger and Vulnerability appear across services

```
Shared pattern: [root cause]
  → svc-A: N occurrences (Trigger: X, Vulnerability: Y)
  → svc-B: M occurrences (Trigger: X, Vulnerability: Z)
```

### 2b — Cascade Patterns
Identify investigations where different services share overlapping `date` + inferred time windows.
A **cascade** = svc-A failed AND svc-B was investigated within the same UTC day.

For each cascade detected:
- Count frequency: how many times has this pair co-occurred?
- Note which service's failure appeared first (by date)
- Note if one service's upstream_causes or downstream_affected mentions the other

### 2c — Shared Blind Spots
Union of all `black_boxes` fields across all services.
Blind spots appearing in ≥2 services → "systemic blind spot"

### 2d — Efficiency Comparison
Compare avg `queries_used` across services. Flag outliers (service taking >2× median queries).

## Step 3: Output

Emit to conversation:

```
## CROSS-SERVICE REVIEW: [svc-A, svc-B, ...] — [Total investigations: N] — [UTC timestamp]

### Shared Root Causes
| Root Cause | Services Affected | Total Occurrences | Common Trigger |
|-----------|------------------|-------------------|----------------|
| ...       | svc-A (N), svc-B (M) | N+M           | ...            |

### Cascade Patterns (co-occurrence ≥ 2)
| Pattern | Frequency | Typical Order |
|---------|-----------|---------------|
| svc-A → svc-B | N | svc-A fails first |

### Systemic Blind Spots (appear in ≥ 2 services)
- [blind spot] — seen in: [svc-A, svc-B]

### Efficiency Outliers
- [service] avg queries: N (median: M) — [above/below median]
```

## Step 4: Recommendations

Produce 1–3 actionable observations:
- "Consider adding svc-B as a dependency hypothesis seed when investigating svc-A failures"
- "Shared Trigger [X] across both services — suggest monitoring at the shared upstream layer"
- "Systemic blind spot [Y] — consider adding dedicated instrumentation"

## Notes

- Cross-service patterns are labeled **co-occurrence**, not causation.
- Pre-v0.57 prose-only entries are excluded from cascade detection (no YAML `date` field).
- This flag does NOT query any observability backend. It is historical analysis only.
