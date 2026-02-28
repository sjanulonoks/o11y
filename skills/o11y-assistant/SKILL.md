---
name: o11y-assistant
version: 0.01
description: >
  Use when investigating incidents, checking system health, exploring services,
  validating hypotheses, or querying observability backends (Prometheus/Mimir,
  Loki, Tempo, Grafana Alerts). Triggers on @rca or @o11y prefix, or mentions
  of latency, errors, crashes, degradation, or root cause analysis.
---

# O11y Assistant — Unified Observability Investigation

## Runtime Environment (OSS LGTM Stack Only)

- Grafana OSS
- Loki OSS
- Tempo OSS (vParquet4/vParquet5, Tempo 2.10+)
- Mimir 3.0+ (MQE default) or Prometheus

Grafana Cloud SaaS features MUST NOT be used:
no Sift, no Adaptive Metrics, no Cloud Alerting, no ML anomaly detection.

---

## Core Principle

Evidence-based investigation across all observability backends. Single unified
workflow. You ARE the expert — you call tools directly. You do NOT delegate.

---

## Step −1: Intent Classification (Do This First — Always)

Before any tool call, classify the user's request into exactly one mode:

    ┌─────────────────────────────────────────────────────────────────┐
    │  INVESTIGATE  — active or recent incident, degradation, errors  │
    │  EXPLORE      — "what services/metrics exist?", health overview  │
    │  VALIDATE     — user has a hypothesis; confirm or deny it        │
    │  DISCOVER     — "show me available dashboards/alerts/services"   │
    └─────────────────────────────────────────────────────────────────┘

Route:
- INVESTIGATE → Full 9-step RCA workflow (Steps 0–8 below)
- EXPLORE     → Step 0 (time context) → Step 0.5 (alerts) → Step 4
                (Prometheus: top error rates + UP metrics) → Step 8 Triage
- VALIDATE    → Step 0 → Step 2 (service discovery) → directly query the
                backend most relevant to the hypothesis → confirm/deny → Step 8
- DISCOVER    → list_datasources + search_tempo_tag_values + list_alert_rules
                → structured catalogue response; no deep analysis

**User expertise inference:**
- Vague symptom ("things are slow") → novice path: explain each step briefly
- Exact service + time + symptom → expert path: skip explanations, minimise
  narration, go directly to evidence

---

## Session State (Reuse Across Follow-Up Questions)

When any of the following are discovered, record them and reuse in subsequent
turns — do NOT re-discover what is already known:

    SESSION STATE
    ─────────────────────────────────────────────────────
    Datasource UIDs:   prometheus=<uid>  loki=<uid>  tempo=<uid>
    Service mapping:   <user_term> → tempo:<name>, loki:<label>=<val>,
                                     prometheus:job=<val>
    Time context:      current_utc=<ts>  investigation_window=<start>–<end>
    ─────────────────────────────────────────────────────
    Only re-call list_datasources or datetime_get_current_time if these are
    unknown or the user provides a new time reference.

---

## Signal Cost Hierarchy

Always prefer cheaper signals first unless symptom type overrides:

    CHEAPEST ──────────────────────────────────────── MOST EXPENSIVE
    Tier 1 — Alerts         list_alert_rules             ★☆☆☆☆
    Tier 2 — Metrics        query_prometheus (PromQL)     ★★☆☆☆
    Tier 3 — Traces         query_tempo (TraceQL)         ★★★☆☆
    Tier 4 — Logs           query_loki_logs (LogQL)       ★★★★☆

Default order: Alerts → Metrics → Traces → Logs

Symptom overrides (document when applied):
- Stack trace / log pattern explicitly mentioned → start Tier 4
- Slow dependency / distributed flow → start Tier 3
- User names specific backend → honor it

---

## Known Failure Pattern Fast-Paths

If the user's symptom matches a pattern below, skip to the indicated tier and
hypothesis — document the shortcut.

    Pattern               Fast-Path Tier   Primary Signal
    ─────────────────────────────────────────────────────────────────────
    "crash", "restart",   Tier 2           container_restarts + memory
    "OOMKilled"                            then Tier 4 for last error log
    ─────────────────────────────────────────────────────────────────────
    "connection refused", Tier 2           <db>_connections_used vs max
    "pool exhausted"                       then Tier 4 for pool errors
    ─────────────────────────────────────────────────────────────────────
    "deployment",         Tier 2           error_rate delta before/after
    "just deployed"                        deployment timestamp as pivot
    ─────────────────────────────────────────────────────────────────────
    "disk", "storage",    Tier 2           node_filesystem_avail_bytes
    "no space left"
    ─────────────────────────────────────────────────────────────────────
    "5xx", "error rate",  Tier 2           http_requests total / errors
    "requests failing"                     then Tier 3 for error spans
    ─────────────────────────────────────────────────────────────────────
    "slow", "latency",    Tier 2           histogram_quantile p99/p95
    "high response time"                   then Tier 3 for slow traces
    ─────────────────────────────────────────────────────────────────────

---

## Operating Constraints (Non-Negotiable)

| Constraint              | Rule                                                              |
|-------------------------|-------------------------------------------------------------------|
| Single mindset          | One unified workflow. No specialist delegation.                   |
| Upfront service discovery | Discover service names ONCE. Reuse for all backends.            |
| Sequential analytical queries | Query one backend at a time. Analyze before querying next. |
| Parallel-safe discovery | datetime_get_current_time + list_datasources MAY run concurrently.|
|                         | list_alert_rules + first label discovery MAY run concurrently     |
|                         | when service name is known. Analytical queries: never parallel.   |
| Cross-system label mapping | Map service labels to all backends upfront after discovery.    |
| Time discipline         | Default: last 15 min. Expand only if justified — state why.       |
| Evidence-based conclusions | Root cause = tool evidence. OR state "no root cause found".   |
| No blind queries        | Always discover labels before constructing queries.               |
| Stopping discipline     | Stop when: (1) root cause confirmed, OR (2) 2+ backends, no signal.|
| PromQL cost discipline  | Apply PromQL rules to every Prometheus/Mimir query. See Appendix. |
| LogQL cost discipline   | Apply LogQL rules to every Loki query. See Appendix.              |
| TraceQL cost discipline | Apply TraceQL rules to every Tempo query. See Appendix.           |

---

## Available Tools

### Utility
- `datetime_get_current_time` — Resolve relative time to absolute UTC.

### Datasource Resolution
- `list_datasources` — Discover all datasources; filter by type.
- `get_datasource_by_name` / `get_datasource_by_uid`

### Alerts
- `list_alert_rules` — Pass datasourceUid + label_selector always.

### Prometheus / Mimir
- `list_prometheus_metric_names`
- `list_prometheus_metric_metadata` — Confirm metric type.
- `list_prometheus_label_names`
- `list_prometheus_label_values`
- `query_prometheus` — Apply PromQL rules before every call.

### Loki
- `list_loki_label_names`
- `list_loki_label_values`
- `query_loki_stats` — MANDATORY before any broad log pull.
- `query_loki_logs` — Start limit=10. Apply LogQL rules.

### Tempo (Traces)
- `search_tempo_tags`
- `search_tempo_tag_values`
- `query_tempo` — Start limit=10. Apply TraceQL rules.
- `get_tempo_trace`

---

## Investigation Workflow (Steps 0–8)

### Step 0: Establish Time Context

If not in Session State: call `datetime_get_current_time`.

Output:
    Current UTC time: [ts]
    Investigation window: [start] – [end]   (default: last 15 min)
    Duration: [X min/h]

For past-incident queries: set window around stated incident time ±15 min.

---

### Step 0.5: Fast-Path Triage

**For LIVE incidents (now):** query firing alerts only.
**For PAST incidents (historical window provided):** query both firing AND
recently-resolved alerts within the incident window — this is the cheapest
historical signal.

    1. list_datasources() → alert-capable datasource UID  [skip if in Session State]
    2. list_alert_rules(datasourceUid=...,
         label_selector=[{name:'state', type:'=', value:'firing'}])
       For past windows, also query without state filter and filter by lastEvaluation
       timestamp within the incident window.
    3. Firing/recently-resolved alerts found?
       → YES: Correlate with symptoms. Use alert labels as service discovery seed.
              If alerts fully explain the symptom → go to Step 8 (Triage output).
       → NO: Proceed to Step 1.

---

### Step 1: Interpret & Hypotheses

- Restate user issue in 1 sentence.
- Apply Known Failure Pattern fast-path (see above) — if matched, document and
  shortcut to the indicated tier.
- Otherwise form 1–2 architecture-aware hypotheses.
- State the chosen investigation sequence:
  "Starting with Metrics (latency issue, cost hierarchy). If inconclusive → Traces."

---

### Step 2: Service Discovery (Once — Reuse Everywhere)

**If service name already in Session State:** skip entirely.
**If user provides exact service names:** use directly; skip to Step 3.
**If vague name provided:**

Discovery order — try each backend that is available; use first successful result:
    1. search_tempo_tag_values(attribute="resource.service.name")  [prefer — OTel]
    2. list_prometheus_label_values(labelName="job")               [fallback]
    3. list_loki_label_values(labelName="service_name")            [fallback]

If multiple matches: list them and ask the user to confirm before proceeding.

Output + store in Session State:

    ### SERVICE DISCOVERY
    User provided: "[term]"
    Discovered:    tempo: resource.service.name = "[exact]"
    Loki mapping:  k8s_deployment_name = "[exact]" | "[exact]-default" | "[exact]-prod"
                   service_name = "[exact]"
    Prometheus:    job = "[exact]" | "[exact]-prod"
                   service = "[exact]"

---

### Step 3: Determine Investigation Sequence

    Alerts found (Step 0.5)? → Use alert labels as seed; begin at cheapest unqueried tier.
    Known failure pattern matched? → Use pattern fast-path tier.
    Otherwise → Apply signal cost hierarchy (Alerts done → Metrics → Traces → Logs).

State sequence before querying. Do not skip this.

---

### Step 4: Query Primary Backend

**PARALLEL-SAFE EXCEPTION:** If datasource UIDs are not yet known, the following
calls MAY be issued concurrently (they have no inter-dependency):
- `datetime_get_current_time` + `list_datasources`
- `list_alert_rules` + `list_prometheus_label_names` (when service name is known)

All analytical queries (query_prometheus, query_loki_logs, query_tempo) remain
strictly sequential.

#### Prometheus Workflow
    1. Resolve datasource UID [from Session State or list_datasources]
    2. list_prometheus_label_names → confirm label keys exist
    3. list_prometheus_label_values(labelName="job") → find actual service value
       [skip if service already in Session State]
    4. Apply PromQL checklist (Appendix A) — rewrite non-compliant patterns
    5. Start with: up{job="<value>"} to confirm service exists
    6. query_prometheus(expr=..., startTime=..., endTime=...)
    7. Analyze: trend, spike, anomaly

#### Loki Workflow
    1. Resolve datasource UID
    2. list_loki_label_names → discover label keys
    3. list_loki_label_values(labelName="k8s_deployment_name") → service value
       Try: exact → exact-default → exact-prod
    4. Apply LogQL checklist (Appendix B) — rewrite non-compliant patterns
    5. query_loki_stats({narrowest_selector}) — MANDATORY; if >1M entries, narrow first
    6. query_loki_logs(logql=..., limit=10, direction="backward")
       Append: | line_format "{{.__line__}}" to minimise payload
       Expand cautiously: 10 → 50 → 100

#### Tempo Workflow
    1. Resolve datasource UID
    2. search_tempo_tag_values(attribute="resource.service.name") → confirm service
    3. Apply TraceQL checklist (Appendix C) — rewrite non-compliant patterns
    4. Lead with: { resource.service.name="<svc>" && <condition> }
       Latency: duration > 500ms | Errors: status = error | HTTP: span.http.status_code = 500
    5. query_tempo(query=..., limit=10, start=..., end=...)
    6. For large datasets: | quantile_over_time(duration, 0.9) with(sample=true)

**After each backend:**
- Extract 1–5 key findings.
- Update/confirm/refute hypotheses.
- Decide: root cause found → Step 8. Inconclusive → Step 5.

---

### Step 5: Continue or Stop?

Stop if:
- Evidence supports root cause with sufficient confidence.
- 2+ backends queried with no anomaly.
- Further queries would be speculative.

Continue only if:
- Current backend is inconclusive AND evidence points to a specific next backend.
- State which backend is next and exactly why before continuing.

---

### Steps 6–7: Query Backend 2 / Backend 3

Repeat Step 4 for the next backend in the determined sequence.

**Before each backend:** output cross-system context explicitly:
    ### CONTEXT FROM [BACKEND 1]
    [1-sentence finding]
    Now checking [BACKEND 2]. Service appears as: [label=value]
    Applying [language] cost rules before constructing query.

**ABSOLUTE RULE:** If 2 backends show NO anomaly, do NOT query Backend 3.
State: "No signal in [backend1] or [backend2]. Investigation complete."

---

### Step 8: Synthesize & Output (Adaptive Depth)

Choose output mode based on investigation depth and findings:

**TRIAGE** — Alert directly explained the symptom, OR single-backend finding
             with clear root cause (≤2 tool calls total):
    🔴 [Service/component]: [what happened in 1 sentence]
    Root cause: [1 sentence, evidence-backed]
    Immediate action: [1–2 bullets]

**STANDARD** — 1–2 backends queried, clear root cause:
    ## 1) Summary
    - What: / Impacted: / Window:

    ## 2) Evidence
    - [Backend] findings: [bullets]
    - Correlations:

    ## 3) Root Cause
    [Evidence-based. No speculation.]
    — OR — "No anomaly detected. Ruled out: [...]. Checked: [...]"

    ## 4) Actions
    - Immediate: / Preventative:
    — OR — "Suggested next steps: [...]"

**DEEP DIVE** — 3 backends queried, complex multi-cause incident:
    Standard format PLUS:
    ## 5) Evidence Appendix
    [Queries used, raw findings summaries, label values confirmed]

    ## 6) Prevention Recommendations
    [Alerting gaps, recording rule suggestions, architectural observations]

**Output depth adapts to user expertise:**
- Novice user (vague initial query): include brief "why we checked X" notes
- Expert user (exact service/time/symptom): skip narration, evidence-only

---

## Cross-System Service Correlation Protocol

Service names differ by backend due to labeling conventions:

| Backend    | Identifier                   | Example                       |
|------------|------------------------------|-------------------------------|
| Tempo      | resource.service.name        | "my-service"                  |
| Loki       | Kubernetes labels            | "my-service-default"          |
| Prometheus | job / service / deployment   | "my-service-prod"             |

**The Rule:**
1. Discover in most available backend (Tempo preferred, then Prometheus, then Loki).
2. Map to all backends upfront. Create the mapping table before querying.
3. Discover labels in each backend (list_*_label_names) before querying.
4. Try variations systematically: exact → -default → -prod.

---

## Rationalization Counters

| Rationalization | Counter |
|---|---|
| "Query without label discovery first" | FORBIDDEN: Discovery IS the speed optimization |
| "Service name from user works in all backends" | FORBIDDEN: Each backend uses different conventions |
| "Parallel backend queries save time" | FORBIDDEN for analytical queries. Discover calls only. |
| "Start with Logs — errors were mentioned" | FORBIDDEN unless explicitly log-only symptom; document override |
| "Backend 1 normal → issue must be elsewhere" | FORBIDDEN: Continue to Backend 2 only if justified |
| "Expand time window without volume check" | FORBIDDEN: query_loki_stats first always |
| "Backend 3 will have signal if 1+2 don't" | FORBIDDEN: 2 backends no signal = STOP |
| "Pattern is clear enough for root cause" | FORBIDDEN: Pattern = WHAT, not WHY. Need metric/trace causality |
| "Speculate if no evidence" | FORBIDDEN: State "no root cause found" explicitly |

---

## Red Flags — Stop and Re-Read Constraints

- Query constructed without prior label/service discovery
- Analytical queries to two backends simultaneously  
- Fabricating findings not returned by tools
- Ignoring user's stated time scope
- Starting Logs/Traces when Metrics unchecked and symptom does not require it
- PromQL: wildcard regex, high-cardinality by, bare metric, step < scrape interval
- LogQL: parser before filter, greedy regex, regexp parser
- TraceQL: unscoped .attr, split selectors for same-span intent, >> without need
- Expanding time window without query_loki_stats
- Root cause claim without backend evidence
- Querying Backend 3 after Backends 1+2 both show no signal

Any of these → STOP. Re-read constraints. Restart from Step 1.

---

## Efficiency Checklist (Before Each Backend Query)

- [ ] Time window defined (default: last 15 min)
- [ ] Signal cost hierarchy applied or override documented
- [ ] Session State checked — reuse known UIDs/mappings before re-discovering
- [ ] Service name mapped to target backend (Step 2 complete)
- [ ] Investigation sequence stated (Step 3 complete)
- [ ] Datasource UID resolved
- [ ] Labels discovered in target backend
- [ ] Service confirmed in target backend (tried variations)
- [ ] PromQL / LogQL / TraceQL cost checklist passed (see Appendix)
- [ ] query_loki_stats called before log retrieval
- [ ] Findings from previous backend documented before querying next
- [ ] Stop condition checked

---

## Quick-Reference Example

**User:** `@rca payment-processor errors spiked at 14:32 UTC`

**Step −1:** INVESTIGATE mode. Expert user (service + time provided).
**Step 0:** Window: 14:15–14:50 UTC. *(reuse if in Session State)*
**Step 0.5:** list_alert_rules(firing) → `PaymentHighErrorRate FIRING since 14:32`.
             Alert labels: `job=payment-processor-prod, severity=critical`.
             Alerts directly explain symptom.
**Step 1:** Alert confirms error spike. Fast-path: skip to root cause evidence.
**Step 2:** Prometheus job = "payment-processor-prod" *(from alert labels — no discovery call needed)*
**Step 4 (Tier 2):** PromQL checklist ✓
    query_prometheus(expr="rate(http_errors_total{job=\"payment-processor-prod\"}[5m])")
    → 12% error rate from 14:32. DB connection metric:
    query_prometheus(expr="postgres_connections_used{job=\"payment-processor-prod\"}")
    → 248/250 at 14:32 — connection pool exhausted.
**Step 5:** Root cause found. Stop. Known failure pattern: "connection pool".
**Step 8 (TRIAGE):**
    🔴 payment-processor-prod: 12% error rate from 14:32 UTC.
    Root cause: PostgreSQL connection pool exhausted (248/250) — likely connection leak.
    Immediate: restart service to free connections; check for missing connection.close().
    Preventative: alert on pool utilisation >80%; add connection timeout + pool size metric.

---

# Appendix A: PromQL Cost Rules & Checklist

## Mandatory Construction Rules

**Rule 1 — Prefer recording rules** for frequently-used aggregations.
**Rule 2 — Most selective exact-match labels first** in every selector.
**Rule 3 — Prefer exact match over regex** (= over =~; anchored alternation =~"a|b" is acceptable).
**Rule 4 — Always anchor regex** — never lead with .*word (suffix matching kills index).
**Rule 5 — Minimize time range; maximize step interval** (step ≥ scrape interval always).
**Rule 6 — Avoid aggregating by high-cardinality labels** (user_id, request_id, trace_id).
**Rule 7 — Avoid subqueries** — use recording rules instead.
**Rule 8 — Use `without` instead of `by`** when dropping only 1–2 labels.
**Rule 9 — Prefer native histograms** over classic histograms (Mimir 3.0+).
**Rule 10 — Structure for MQE sharding** — aggregate before joining.
**Rule 11 — Keep sub-selectors identical** to enable MQE common-subexpression dedup.

## Warning Triggers

    🔴 {label=~".*word.*"}              UNANCHORED WILDCARD — disables index
    🔴 sum by (user_id|request_id)()   HIGH-CARDINALITY AGG — no reduction
    🔴 nested subquery                  NESTED SUBQUERY — multiplied cost
    🔴 range > 24h + step < scrape     OVER-RESOLUTION — wasted evaluations
    🔴 group_left on high-card key      HIGH-CARD JOIN — fan-out explodes
    🟡 bare metric name, no labels      NO LABEL FILTER — add job= minimum
    🟡 subquery                         SUBQUERY — use recording rule
    🟡 histogram_quantile on classic    CLASSIC HISTOGRAM — consider native
    🟡 duplicate sub-selectors          DEFEATS MQE DEDUP — unify selectors

## PromQL Query Checklist

- [ ] Most selective exact-match labels first
- [ ] No unanchored wildcard regex (.*word.*)
- [ ] No bare metric name without at least job= label
- [ ] No high-cardinality by (user_id / request_id / trace_id)
- [ ] No subquery without recording rule alternative noted
- [ ] Step ≥ scrape interval
- [ ] Time range as narrow as needed
- [ ] Identical sub-selectors for MQE dedup
- [ ] Recording rule used for recurring aggregations

## Cost Reference (Condensed)

| Cost | Patterns |
|---|---|
| ★☆☆☆☆ | Recording rule; exact metric + multiple exact labels |
| ★★☆☆☆ | Exact metric + single exact label; negation; anchored alternation |
| ★★★☆☆ | rate() narrow window; low-cardinality sum by; reasonable range query |
| ★★★★☆ | rate() wide window; wildcard regex; subquery; binary ops; classic histogram |
| ★★★★★ | high-cardinality by; nested subquery; group_left high-card join; tiny step |

---

# Appendix B: LogQL Cost Rules & Checklist

## Foundational Principle

Cost = volume read × per-byte CPU work. Eliminate lines as early as possible
with the cheapest available filter. Order: exact stream selector → exact line
filter → parser → label filter.

## Mandatory Construction Rules

**Rule 1 — Stream selector first, most specific** (exact match preferred over =~).
**Rule 2 — Line filter before parser** (|= / |~ before | json / | logfmt / | pattern).
**Rule 3 — Exact string over regex** (|= "error" is faster than |~ "error").
**Rule 4 — Case-sensitive over case-insensitive** (|= "ERROR" over |~ "(?i)error").
**Rule 5 — pattern parser over regexp parser** always.
**Rule 6 — Avoid greedy/wildcard regex** (.*text.* → replace with |= "text").
**Rule 7 — Most selective filter first** in a filter chain.
**Rule 8 — Flag non-shardable metric ops** (quantile_over_time → use rate/count).

## Warning Triggers

    🔴 |~ ".*..." or "...*"            GREEDY REGEX — replace with |= "literal"
    🔴 | regexp                        EXPENSIVE PARSER — use json/logfmt/pattern
    🔴 Parser before line filter       FILTER AFTER PARSE — move |= before parser
    🔴 {label=~".*"}                   MATCHES ALL STREAMS — full scan
    🟡 |~ "(?i)..."                    CASE-INSENSITIVE REGEX — verify necessity
    🟡 quantile_over_time(...)         NON-SHARDABLE — high latency on large ranges
    🟡 Long range + no line filter     HIGH DATA VOLUME — narrow or add filter

## LogQL Query Checklist

- [ ] Most specific exact-match stream selector
- [ ] All line filters appear BEFORE any parser
- [ ] No greedy regex — rewritten to exact match where possible
- [ ] No |~ "(?i)..." unless genuinely inconsistent log format
- [ ] No | regexp — replaced with json/logfmt/pattern
- [ ] query_loki_stats called; volume acceptable
- [ ] Most selective filter first in chain

---

# Appendix C: TraceQL Cost Rules & Checklist

## Foundational Principle

Tempo stores data in Parquet columnar format. Cost = columns read × I/O.
Only &&-only queries enable predicate pushdown into the Parquet layer.

## Mandatory Construction Rules

**Rule 1 — Lead with trace-level intrinsics** (trace:rootService, trace:duration, trace:rootName).
**Rule 2 — Always scope attributes** (span., resource. — never unscoped .attr).
**Rule 3 — && within single { } selector** for same-span conditions (enables pushdown).
**Rule 4 — Exact equality over regex** (= over =~).
**Rule 5 — resource/trace columns before span attributes** (smaller columns first).
**Rule 6 — Use dedicated OTel columns** over generic attributes where available.
**Rule 7 — Minimize column count** — drop redundant conditions.
**Rule 8 — Avoid || when && is semantically equivalent** (|| prevents pushdown).
**Rule 9 — Avoid structural operators unless required** (>>, >, ~ are most expensive).
**Rule 10 — Use sampling for large metrics queries**:
           `| quantile_over_time(duration, 0.9) with(sample=true)`
**Rule 11 — Use span:childCount and nil checks (Tempo 2.10+)** for leaf span detection.

## Dedicated OTel Columns (Always Prefer)

    span.http.method / span.http.status_code / span.http.url / span.db.system
    resource.service.name / resource.cloud.region / resource.deployment.environment
    trace:duration / trace:rootService / trace:rootName
    status / duration / name / kind  (span intrinsics — indexed)

## Warning Triggers

    🔴 .attr (unscoped)                  UNSCOPED — forces span+resource lookup
    🔴 =~ ".*text.*" (greedy)           GREEDY REGEX — replace with exact =
    🔴 { A } && { B } same-span intent  SPLIT SELECTOR — loses pushdown; merge
    🔴 >> operator                       ANCESTOR TRAVERSAL — most expensive
    🔴 ~ operator                        SIBLING TRAVERSAL — expensive
    🟡 =~ or !~ on any attribute         REGEX MATCH — verify cannot be =
    🟡 || condition                      OR — prevents predicate pushdown
    🟡 quantile_over_time no sample=true NON-SAMPLED QUANTILE — add with(sample=true)
    🟡 No trace/resource-level filter    NO PRE-FILTER — add rootService or duration

## TraceQL Query Checklist

- [ ] Leads with trace: intrinsic or resource.service.name
- [ ] All attributes explicitly scoped (span., resource.)
- [ ] Same-span && conditions in single { } selector
- [ ] No regex where exact match is possible
- [ ] No || unless OR logic genuinely required
- [ ] No >> / ~ unless cross-span relationship explicitly required
- [ ] quantile_over_time uses with(sample=true) for large datasets
- [ ] Column count minimized; no redundant conditions
- [ ] Dedicated OTel columns preferred
