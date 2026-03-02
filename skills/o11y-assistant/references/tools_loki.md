# Loki Tools & Cost Rules

> Loaded by SKILL.md at Step 4 when querying log backends.
> Covers: Tool parameters, backend-specific query notes, LogQL cost rules.

---

## Tool Reference

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `list_loki_label_names` | `datasourceUid`, time range (optional) | Discover log stream label keys. | Different Loki instances have different label sets. |
| `list_loki_label_values` | `datasourceUid`, `labelName`, time range (optional) | Discover values for label | 🟡 High-cardinality warning: Filter to higher-level label first (namespace, cluster) before high-cardinality labels |
| `query_loki_stats` | `datasourceUid`, `logql` (selector only), time range | **MANDATORY before broad log pull.** Returns: `streams`, `chunks`, `entries`, `bytes`. Non-existent selectors return all zeros (no error). | Validate selector AND estimate volume (fast ~100MB vs slow >1GB) |
| `query_loki_logs` | `datasourceUid`, `logql`, `limit` (default 10, max 100), `direction` ("backward"/"forward"), `queryType` ("instant"/"range"), time range | Execute LogQL. Start limit=10. | 🔴 **Timestamp formats differ:** Log entries = quoted nanosecond strings. Metrics = Unix seconds. Empty results return hints (not error). |
| `query_loki_patterns` | `datasourceUid`, `logql` (stream selector only), time range | Detect common log patterns (anomaly detection) | Empty array = logs follow single format (not error) |
| `search_logs` | `DatasourceUID`, `Pattern`, `Start`, `End`, `Limit` (optional, max 1000) | Quick text/regex search across Loki or ClickHouse. Auto-generates queries, escapes special chars. Paginated results with hints. | Pattern auto-detection: simple text = literal, regex chars = regex. Do NOT manually escape. |

---

## Backend-Specific Query Notes

- **Volume check first:** `query_loki_stats({selector})` MANDATORY before broad log pull. If >1M entries, narrow selector first.
- **Payload optimization:** Append `| line_format "{{.__line__}}"` to minimize payload
- **Expand limit cautiously:** 10 → 50 → 100
- **Direction:** "backward" (newest first) for "what just happened"; "forward" (oldest first) for chronological sequence
- **Anomaly detection:** `query_loki_patterns({selector})` for pattern detection (optional)

---

## LogQL Cost Rules (Appendix B)

### Foundational Principle

Cost = volume read × per-byte CPU work. Eliminate lines as early as possible with cheapest filter.
Order: exact stream selector → exact line filter → parser → label filter.

### Mandatory Construction Rules

1. **Stream selector first, most specific** (exact match preferred over `=~`)
2. **Line filter before parser** (`|=` / `|~` before `| json` / `| logfmt` / `| pattern`)
3. **Exact string over regex** (`|= "error"` faster than `|~ "error"`)
4. **Case-sensitive over case-insensitive** (`|= "ERROR"` over `|~ "(?i)error"`)
5. **pattern parser over regexp parser** always
6. **Avoid greedy/wildcard regex** (`.*text.*` → replace with `|= "text"`)
7. **Most selective filter first** in filter chain
8. **Flag non-shardable metric ops** (quantile_over_time → use rate/count)

### Warning Triggers

```
🔴 |~ ".*..." or "...*"            GREEDY REGEX — replace with |= "literal"
🔴 | regexp                        EXPENSIVE PARSER — use json/logfmt/pattern
🔴 Parser before line filter       FILTER AFTER PARSE — move |= before parser
🔴 {label=~".*"}                   MATCHES ALL STREAMS — full scan
🟡 |~ "(?i)..."                    CASE-INSENSITIVE REGEX — verify necessity
🟡 quantile_over_time(...)         NON-SHARDABLE — high latency on large ranges
```

### LogQL Query Checklist

- [ ] Most specific exact-match stream selector
- [ ] All line filters appear BEFORE any parser
- [ ] No greedy regex — rewritten to exact match where possible
- [ ] No `|~ "(?i)..."` unless genuinely inconsistent log format
- [ ] No `| regexp` — replaced with json/logfmt/pattern
- [ ] `query_loki_stats` called; volume acceptable
- [ ] Most selective filter first in chain
