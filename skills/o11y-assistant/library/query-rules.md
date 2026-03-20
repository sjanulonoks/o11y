# Query Language Cost Rules

> Apply the checklist for the target backend before constructing each analytical query at Step 4.

---

## Appendix A: PromQL Cost Rules

1. **Most selective exact-match labels first** in every selector
2. **Prefer exact match over regex** (= over =~; anchored alternation =~"a|b" acceptable)
3. **Always anchor regex** — Lead regex ALWAYS with explicit anchors (avoid `.*word` to preserve index).
4. **Minimize time range; maximize step interval** (step ≥ scrape interval)
5. **Avoid high-cardinality labels** (user_id, request_id, trace_id) in `by` aggregations
6. **Use `without` instead of `by`** when dropping only 1–2 labels

7. **Use `increase()` or `count_over_time()` for exact counts;** rate gives per-second average, not totals.

```
🔴 {label=~".*word.*"}              UNANCHORED WILDCARD — disables index
🔴 sum by (user_id|request_id)()   HIGH-CARDINALITY AGG — no reduction
🔴 rate(m[5m]) * 300               RATE×WINDOW — use increase() or count_over_time()
🟡 bare metric name, no labels      NO LABEL FILTER — add service-identity label minimum
```

**Checklist:** Labels first ✓ No unanchored regex ✓ No bare metric ✓ No high-cardinality by ✓ Step ≥ scrape ✓ No rate×window for counts ✓

---

## Appendix B: LogQL Cost Rules

Cost = volume read × per-byte CPU. Eliminate lines early with cheapest filter.

1. **Stream selector first, most specific** (exact match preferred)
2. **Line filter before parser** (`|=` before `| json` / `| logfmt`)
3. **Exact string over regex** (`|= "error"` faster than `|~ "error"`)
4. **pattern parser over regexp parser** always
5. **Most selective filter first** in filter chain

```
🔴 |~ ".*..." or "...*"            GREEDY REGEX — replace with |= "literal"
🔴 | regexp                        EXPENSIVE PARSER — use json/logfmt/pattern
🔴 Parser before line filter       FILTER AFTER PARSE — move |= before parser
🔴 {label=~".*"}                   MATCHES ALL STREAMS — full scan
```

**Checklist:** Specific selector ✓ Line filter before parser ✓ No greedy regex ✓ `query_loki_stats` called ✓

---

## Appendix C: TraceQL Cost Rules

Tempo stores data in Parquet. Cost = columns read × I/O. Only &&-only queries enable pushdown.

1. **Lead with trace-level intrinsics** (trace:rootService, trace:duration)
2. **Always scope attributes** (span., resource. — MUST be scoped)
3. **&& within single { } selector** for same-span conditions (enables pushdown)
4. **Exact equality over regex** (= over =~)
5. **resource/trace columns before span attributes** (smaller columns first)
6. **Use dedicated OTel columns** (span.http.method, resource.service.name, etc.)
7. **Avoid || when && semantically equivalent** (|| prevents pushdown)
8. **Avoid structural operators unless required** (>>, >, ~ most expensive)

```
🔴 .attr (unscoped)                  UNSCOPED — forces span+resource lookup
🔴 =~ ".*text.*" (greedy)           GREEDY REGEX — replace with exact =
🔴 { A } && { B } same-span intent  SPLIT SELECTOR — merge into single { }
🔴 >> operator                       ANCESTOR TRAVERSAL — most expensive
```

**Checklist:** Scoped attributes ✓ Single { } for same-span ✓ No unscoped attrs ✓ No >> unless needed ✓
