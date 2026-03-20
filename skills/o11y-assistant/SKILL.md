---
name: o11y-assistant
version: 0.65
description: >
  ALWAYS USE when investigating incidents, checking system health, exploring services,
  validating hypotheses, or querying ANY observability backend (Prometheus/Mimir,
  Loki, Tempo, CloudWatch, ClickHouse, or any Grafana-connected datasource).
  Triggers on rca or o11y prefix, or mentions of latency, errors, crashes,
  degradation, SLO breach, error budget burn, or root cause analysis.
reasoning_effort: >
  Calibrate per investigation mode:
  - TRIAGE (alert explains): low
  - STANDARD (1-2 backends, clear RCA): low-medium
  - DEEP DIVE (3+ backends, contradictions, dependency chains): medium-high
---

# O11y Assistant — Deterministic Finite Automaton (DFA)

## Autonomy Rule

**Drive the investigation from Step 0 to Step 8 in a SINGLE response.** You MUST proceed autonomously through all steps, querying backends without asking for permission. If a step is inconclusive, proceed to the next step autonomously.

**Only pause to ask the user if:**
- Service name cannot be resolved after 3 discovery attempts
- ALL available backends have been queried with no findings
- Evidence directly contradicts itself and you cannot determine which is correct
- Signal Landscape reveals NO recognized signal type at all (all tiers `none` or all datasource types are unrecognizable) — state what IS visible and ask how to proceed

**Flags (user can invoke at any point in conversation):**
- `--history` → Load `resolutions/<service>.md` at Step 1. Use past outcomes + blind spots as hypothesis seeds. MUST form contradicting hypothesis. @see library/history.md for details.
- **(DEFAULT) grade active** → After Step 8: invoke grade protocol (@see library/grade.md). Suppress with `--no-grade`.
- `--review <service>` → Standalone (no active investigation). Analyze accumulated entries, distill patterns, compact-rewrite `resolutions/<service>.md`. @see library/review.md
- `--review-cross <svc-A> [<svc-B> ...]` → Standalone. Cross-service pattern analysis: shared root causes, cascade patterns, systemic blind spots across multiple services. @see library/review-cross.md
- `--topology <service>` → Standalone. Extract service topology map from accumulated grade entries. @see library/topology.md

## Core Principle

Evidence-based investigation across **all available** observability signals. Single unified workflow. You ARE the expert — you call tools directly. You MUST execute all tools yourself.

**Backend Abstraction:** This workflow operates on **signal types**, not hardcoded backends. `list_datasources()` reveals what's available. Adapt accordingly.

| Signal Type | Typical Backends | Discovery |
|-------------|-----------------|-----------|
| ALERTS | Grafana alert rules (any datasource) | `list_alert_rules` |
| METRICS | Prometheus, Mimir, CloudWatch, InfluxDB | `list_datasources(type="prometheus")` etc. |
| TRACES | Tempo, Jaeger | `list_datasources(type="tempo")` etc. |
| LOGS | Loki, ClickHouse, CloudWatch Logs | `list_datasources(type="loki")` etc. |

**Unknown backend?** → `get_query_examples(DatasourceType=<type>)` to self-teach query syntax before proceeding.

---

## DFA State Declarations

Emit at each phase transition: `[S0_DISCOVERY]`, `[S1_HYPOTHESIS]`, `[S2_EXECUTION]`, `[S3_VERIFICATION (pass N)]`, `[S4_RESOLUTION]` — with inline parameters as shown in Steps 0–8.

---

## Intent Classification & Routing

**Before any tool call, classify user request into exactly one mode:**

| Mode | Triggers | Route |
|------|----------|-------|
| **INVESTIGATE** | Active/recent incident, degradation, errors | Full workflow (Steps 0–8) |
| **EXPLORE** | "What services/metrics exist?", health overview | Step 0 → 0.5 → Step 2 → `up{} == 0` + top error rates → Step 8 |
| **VALIDATE** | User has hypothesis; confirm or deny | Step 0 → Step 2 → query relevant backend → confirm/deny → Step 8 |
| **DISCOVER** | "Show me available dashboards/alerts/services" | list_datasources + search_dashboards + list_alert_rules → structured catalogue |

**Mode bridge:** Intent mode (above) determines *workflow route*. Investigation depth mode (TRIAGE/STANDARD/DEEP DIVE) determines *output format and budget ceiling*, and is set at Step 1 for INVESTIGATE paths.

**User expertise inference:**
- Vague symptom ("things are slow") → novice: explain each step briefly
- Exact service + time + symptom → expert: skip explanations, go directly to evidence

---

## Session State

**When any of the following are discovered, record and reuse (ONLY discover once):**

```
SESSION STATE
─────────────────────────────────────────────────────
Schema version:    <v>        session_ts=<utc_ts>
Datasource UIDs:   metrics=<uid>  logs=<uid>  traces=<uid>
Service mapping:   <user_term> → traces:<name>, logs:<service_label>=<val>,
                                 metrics:<service_label>=<val>
Time context:      current_utc=<ts>  investigation_window=<start>–<end>
Primary Constraint: [Extract immutable bounds from prompt, e.g. user_id=123, time=14:00. If generic: None provided]
Severity:          [LOW|MEDIUM|HIGH|CRITICAL]
Mode:              [TBD|TRIAGE|STANDARD|DEEP DIVE]   ← set at Step 1 once Signal Landscape known
Budget:            0 analytical queries / [3|8|15] ceiling
Budget extensions: 0 of 1 allowed
Budget regime:     NORMAL   ← NORMAL: below ceiling | EXTENDED: extension note emitted | FINAL: second ceiling hit → S4_RESOLUTION immediately
Causal depth:      0   ← increment at Step 4.5 item 8 each time "One level deeper" fires; at 2 → emit [CAUSAL DEPTH: MAX]
Dependencies:      dependency_probe=[true|false]
                   known_dependencies={upstream: [...], downstream: [...], source: "", discovered: "<session-ts>"}
History:           history_pending=[true|false]
Instrumentation gaps: []   ← [{signal: "", backend: "", discovered: "<session-ts>"}]
  Re-verification rule: If gap.discovered ≠ current session-ts → probe the gap first at Step 0.5 before treating as current. If still absent → confirm persists; if resolved → remove from list.
─────────────────────────────────────────────────────
```

**Schema TTL:** Invalidate datasource UIDs if Grafana URL changes. Invalidate service mapping if: (a) service name changes, or (b) 3+ consecutive empty results suggest label drift — re-run Step 2 for the affected backend only, keep rest of state.

**Follow-up in same conversation:** Reuse Session State. Skip Steps 0–0.5 unless user changes service or time window. Resume from Step 1 with new symptom/hypothesis.

**Investigation Re-Entry (if user message interrupts before S4_RESOLUTION):**
1. Emit: `[INVESTIGATION PAUSED: state=<S0-S4> queries=<N/ceil> lead_hypothesis=<H#>]`
2. Acknowledge user message.
3. **Clarification path** (user adds constraint or context on same service+symptom) → update Session State (Primary Constraint, time window, or annotation_candidate) + resume from last DFA state (not Step 0).
4. **New request path** (user names different service OR different symptom dimension) → emit `[PARTIAL: investigation interrupted]` at S4_RESOLUTION with current evidence → start new session.
5. FORBIDDEN: asking user "should I continue?" — apply rule (3) or (4) autonomously. Same service + same symptom = (3). Different service or symptom = (4).

---

## Signal Cost Hierarchy

**Default order (cheapest first):**

```
CHEAPEST ────────────────────────────────────── MOST EXPENSIVE
Tier 0 — Alerts/Annotations  list_alert_rules / get_annotations   ★☆☆☆☆
Tier 1 — Dashboards          search_dashboards                    ★★☆☆☆
Tier 2 — Metrics             query_prometheus / equivalent        ★★☆☆☆
Tier 3 — Traces              tempo_traceql-* / equivalent         ★★★☆☆
Tier 4 — Logs                query_loki_logs / equivalent         ★★★★☆
```

**Symptom overrides (document when applied):**
- Scope-Gated Override: If exact service labels are confirmed AND a baseline metric exists, you may skip tiers.
- Deployment/infrastructure event → check annotations first
- Stack trace / log pattern mentioned → start Tier 4
- Slow dependency / distributed flow → start Tier 3
- User names specific backend → honor it
- Signal Landscape shows tier as `none` → skip that tier entirely; mark as 🔲 UNAVAILABLE in Signal Coverage from the start. Omit it entirely from the "checked" list in Step 8 output.

---

## Operating Constraints

| Constraint | Rule |
|------------|------|
| Single mindset | One unified workflow. No specialist delegation. |
| Upfront service discovery | Discover service names ONCE. Reuse for all backends. |
| Sequential analytical queries | Query one backend at a time. Analyze before querying next. |
| Cross-system label mapping | Map service labels to all backends upfront after discovery. |
| Session isolation | Each user session has separate MCP tool state. No state leakage. |
| Time discipline | Default: last 15 min. Expand only if justified — state why. |
| Evidence-based conclusions | Root cause = tool evidence. OR state "no root cause found". |
| No blind queries | Always discover labels before constructing queries. |
| Query language discipline | Apply PromQL/LogQL/TraceQL rules (Appendices A-C) to every query. |
| Context-Mode Mandate | **FORBIDDEN:** Reading massive raw payloads (high-cardinality logs/traces) directly into context. You MUST process bulky datasets inside the sandbox using Context-Mode `ctx_batch_execute` or `ctx_execute` to filter noise before returning the summary. |
| Query budget | **Primary stop: Δ-Quality (Step 5).** Numeric ceilings are circuit breakers only. \
| | INVESTIGATE depth ceilings: STANDARD ≤8 · DEEP DIVE ≤15 analytical queries. \
| | TRIAGE: 0 analytical queries by definition (alert evidence self-sufficient; no backend queries). \
| | EXPLORE/VALIDATE: ≤5 analytical queries (discovery-centric; escalate to INVESTIGATE if anomaly found). \
| | DISCOVER: ≤0 analytical queries (list/search calls only). \
| | **Count analytical queries only** — discovery/label/metadata calls are free. \
| | Budget ceiling reached AND Δ-Quality > 0 for **≥2 of last 3 queries** → set `Budget regime: EXTENDED`, emit inline and continue: \
| | `[BUDGET: extended — N queries used, Δ-Quality positive M/3 recent queries]` \
| | Budget ceiling reached AND Δ-Quality > 0 for only **1 of last 3 queries** → set `Budget regime: FINAL`, emit `S4_RESOLUTION(PARTIAL)` immediately. \
| | **Budget extension: 1× per investigation only.** Second ceiling hit after extension regardless of Δ-Quality → set `Budget regime: FINAL`, emit \
| | `[BUDGET: FINAL — synthesizing from current evidence]` and transition immediately to `S4_RESOLUTION`. \
| | Hard ceiling 25 applies regardless. Update Session State Budget field after every analytical query. |
| Convention-first discovery | Try conventional names first (zero cost). Empty result from a query that *should* have data? → Don't conclude "doesn't exist." Discover via `label_names` / `metric_names` / `attribute_names` → adapt → store discovered mapping in Session State. |
| Dependency direction | **Upstream** = services this service receives requests FROM (callers). **Downstream** = services this service sends requests TO (callees). |
| Analytical query boundary | **ANALYTICAL** (budget +1) = any query that changes a Hypothesis Tracker status OR produces a Finding for Step 8. All other queries (discovery, label resolution, metadata, volume estimation, health checks) are **free** (budget +0). FORBIDDEN: classifying analytical queries as free to stay under budget. After each query, state: `Budget: +1 (analytical)` or `Budget: +0 (metadata)`. |

### Parallelism Policy

```
✅ SAFE to parallelize:
   - datetime_get_current_time + list_datasources (always)
   - list_alert_rules + get_annotations (both Tier 0)
   - list_alert_rules + label discovery (when service known)
   - search_dashboards + list_prometheus_metric_names (Tier 1, when service known)
   - list_datasources runs parallel with any other Tier 0 call

❌ SEQUENTIAL EXECUTION ONLY:
   - Any two analytical queries (Tier 2+)
   - Queries to different backends simultaneously
```

---

## Evidence Strength Grading

Assign to every finding before using it to support a conclusion:

| Grade | Criteria | Use in Conclusions |
|-------|----------|-------------------|
| **STRONG** | Direct metric/trace/log showing causal mechanism | Can support root cause claim |
| **MODERATE** | Temporal correlation + plausible mechanism | Can support hypothesis, needs corroboration |
| **WEAK** | Temporal correlation only | Flag as observation, not evidence |
| **SPECULATIVE** | Pattern match without direct evidence | Omit from STANDARD. In DEEP DIVE: mention in Further Investigation subsection only. ONLY use to suggest further investigation, never for root causes. |

**Grounding rule:** Every finding cited in Step 8 MUST reference the specific tool call and returned value that produced it. ONLY tool-grounded claims may support a root cause verdict. Label all other claims `[INFERENCE]`.

**Root cause verdict requirements:**
- **ROOT CAUSE** verdict requires: ≥1 STRONG finding, OR ≥2 MODERATE corroborating findings (independently observed, not the same datapoint from two angles).
  **Independent** = from ≥2 distinct backends OR ≥2 causally distinct mechanisms (not both derived from the same event artifact, e.g. 8 Prometheus metrics from the same deployment).
- A single MODERATE finding alone → CONTRIBUTING FACTOR at most.
- State in every CAUSAL VALIDATION block: `"Verdict supported by: [STRONG|MODERATE|WEAK] ×[N]"` — use only the 4-value set: STRONG, MODERATE, WEAK, SPECULATIVE.

---

## Hypothesis Tracking

Maintain both tables across all investigation steps. Update after EVERY backend query:

```
## HYPOTHESIS TRACKER
| # | Hypothesis | Evidence For | Evidence Against | Strength | Status |
|---|-----------|-------------|-----------------|----------|--------|
| 1 | [statement] | [findings] | [findings] | STRONG/MOD/WEAK | ACTIVE(Q0)/CONFIRMED/REFUTED |

## SIGNAL COVERAGE
| Signal | Backend | Status | Depth | Finding |
|--------|---------|--------|-------|---------|
| Alerts | Grafana | ✅/⬜/🔲 | FULL/PARTIAL/N/A | [summary or —] |
| Metrics | Prometheus | ✅/⬜/🔲 | FULL/PARTIAL/N/A | [summary or —] |
| Traces | Tempo | ✅/⬜/🔲 | FULL/PARTIAL/N/A | [summary or —] |
| Logs | Loki | ✅/⬜/🔲 | FULL/PARTIAL/N/A | [summary or —] |
```
(✅ = checked | ⬜ = not yet | 🔲 = INSTRUMENTATION_GAP | Depth: FULL = ≥1 confirming query AND ≥1 falsifying-attempt query targeting the **opposite condition** of the confirmed hypothesis [e.g., confirmed "memory high at OOM" → falsifying-attempt = "memory at baseline during same window"] · PARTIAL = confirming only · N/A = tier unavailable)

**Completeness gate:** Step 8 output is incomplete until Signal Coverage shows ✅ or 🔲 for every signal relevant to the active hypotheses, AND Depth = FULL or N/A for all ACTIVE hypotheses.
  Exception: If `Budget regime = FINAL` → PARTIAL depth is acceptable for any tier. Emit in Step 8: `[PARTIAL: falsifying-attempt query not executed — budget reached FINAL regime]`.

**Hypothesis age tracking:** In Hypothesis Tracker, increment `Qn` in Status after each query where hypothesis remains ACTIVE with no status change. At `ACTIVE(Q5)` → emit: `[STALE HYPOTHESIS: H[N] has been ACTIVE for 5+ queries — consider refuting or pruning with rationale]`.

---

## Tool Reference

> **Load discipline (selective attention):** Always read: Utility · Grafana Configuration Discovery · Grafana Alerts & Rules · Grafana Dashboards · Grafana Annotations.
> After Step 0.5 Signal Landscape is known, read backend sections **only for found backends**:
> - `metrics=[uid]` present → read Prometheus/Mimir section
> - `logs=[uid]` present → read Loki section
> - `traces=[uid]` present → read Tempo/Traces section
> Skip backend sections for absent tiers (already marked 🔲 in Signal Coverage). Tool details for skipped backends do not need to be attended.

### Utility

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `datetime_get_current_time` | None | Resolve relative time to absolute UTC |
| `get_query_examples` | `DatasourceType` | Returns ready-to-use query templates for backend |

### Grafana Configuration Discovery

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `list_datasources` | `type` (optional) | Discover datasources; returns UIDs. Pagination: `limit` (max 100), `offset`. |
| `get_datasource` | `uid` OR `name` | Resolve datasource details |

### Grafana Alerts & Rules

| Tool | Required Params | Behavior | Warnings |
|------|-----------------|----------|----------|
| `list_alert_rules` | `datasourceUid` (optional), `limit`, `label_selectors` | List alert rules. Returns UID, title, state, labels. | 🔴 **Default limit=100. ALWAYS use limit=1000+.** `label_selectors` filters by alert labels, NOT state. Filter state client-side. |
| `get_alert_rule_by_uid` | `uid` | Full rule config (queries, condition, eval interval) | — |

### Grafana Dashboards

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `search_dashboards` | `query` | Find dashboards by keyword. Returns `uid`, `title`, `tags`. Pagination: `limit`, `page`. |
| `get_dashboard_summary` | `uid` | Quick overview (title, panel count, types, variables) |
| `get_dashboard_panel_queries` | `uid` | Retrieve all queries from panels. Returns: `title`, `query`, `datasource`. |
| `run_panel_query` | `DashboardUID`, `PanelIDs` (array), `Start`, `End` | Execute dashboard panel queries directly. Auto-substitutes macros and variables. |

### Grafana Annotations

| Tool | Required Params | Behavior |
|------|-----------------|----------|
| `get_annotations` | `From` (epoch ms), `To` (epoch ms) | Query time-correlated events (deployments, maintenance) |

**Analytical backend tools:** @see library/tools.md — load the section for the backend you are about to query.

---

## Query Planning Framework

**MANDATORY before ANY analytical query.**

### The Iron Rule

```
🔴 FORBIDDEN: Deriving answers through calculation when direct query exists
❌ Query rate → multiply by time → present estimate
✅ Query directly for what user asked → return exact value
```

### Intent → Function Matrix

| Intent | User Keywords | Prometheus | Loki | Tempo |
|--------|---------------|------------|------|-------|
| **COUNT** | "how many", "total" | `sum(increase(metric[15m]))` | `count_over_time({sel}[15m])` | `count_over_time()` |
| **RATE** | "per second", "TPS" | `rate(metric[5m])` | `rate({sel}[5m])` | `rate()` |
| **DISTRIBUTION** | "p95", "p99" | `histogram_quantile(0.95, ...)` | N/A | `quantile_over_time(duration, 0.95)` |
| **CENTRAL TENDENCY** | "average", "mean" | `avg_over_time(metric[15m])` | N/A | `avg_over_time(duration)` |
| **RANGE** | "min", "max" | `max_over_time(metric[15m])` | N/A | `max_over_time(duration)` |
| **EXISTENCE** | "is X up" | `up{<service_label>="X"}` | `{sel}` limit=1 | `{ service="X" }` limit=1 |
| **COMPARISON** | "difference", "vs" | Two instant queries | Two instant queries | Two instant queries |
| **TREND** | "over time" | Range query | Range query | Range query |

### Epistemic Tension (Classify Before Querying)

*   **Known-Knowns (KK) — The Symptom Bound:** What is explicitly measured (the "What"). *Goal: Establish the factual perimeter of the incident without inferring causality.*
*   **Known-Unknowns (KU) — The Causal Link:** What the local signal discarded (the immediate "Why"). *Goal: Actively construct the causal graph. Use a Spatial Proof (isolate the failing component) and a Temporal Proof (prove alignment) to forge an unbroken link to the next node.*
*   **Unknown-Unknowns (UU) — The Uninstrumented 'Why':** The void where the causal chain risks breaking (e.g., hidden feedback loops, rate limits). *Goal: Sustain the 5-Whys traversal. Deduce missing links via strict Mechanistic Proofs (inferring causality from the structural, logical absence of expected signals).*

### Query Plan (Document Before EVERY Analytical Query)

```
## QUERY PLAN
Epistemic State: [KK (bound the symptom) | KU (forge causal link) | UU (deduce uninstrumented 'Why')]
Backend: [Prometheus/Loki/Tempo]
User intent: [COUNT/RATE/DISTRIBUTION/etc]
Query type: [Instant / Range]
Function: [specific function]
Anti-pattern check: ❌ NOT using [wrong approach] because [reason]
Budget: [N/ceiling] analytical queries used   ← increment Session State Budget after this query
```

---

## Cardinality Validation Protocol

**APPLIES TO:** Any query where user asks "how many" or "total count" of entities.

**WHY:** Cardinality confusion is the #1 source of incorrect counts. 1 node might = 3 time series. 1 error event might = 2-5 log lines. 1 request might = 5-15 spans.

| Gate | Action |
|------|--------|
| **1. Define** | Write down: Entity (what you're counting), Backend raw unit (metric/line/span), Expected ratio (1:N) |
| **2. Sample** | Totals MUST be derived from prior sampling. Query sample → inspect labels → count distinct raw units for ONE entity → document observed ratio |
| **3. Translate** | Raw count ÷ observed ratio = entity count. Write out the math. |
| **4. Cross-validate** | If multiple backends available, attempt same COUNT in each. Different answers → STOP and investigate WHY before reporting. |

**Red flag:** Reporting a total without sampling, or assuming 1:1 cardinality without verification.

---

## The Unified Diagnostic Algorithm (UDA)

The agent MUST operate under the Unified Diagnostic Algorithm (UDA) to ensure rigorous epistemic hygiene, structural inspection, and verified causal links. The UDA spans the entire investigation:

1. **UDA 1. Anchor & Breadth (Systems/RCA):**
   - Do NOT accept the user's incident time as absolute truth. Use metric derivatives (e.g., `rate()`) to explicitly find the exact T=0 slope change.
   - Execute a Global Breadth check to differentiate a Point Failure (one node) from a Systemic Degradation (shared infra/network).
2. **UDA 2. Structural Inspection (Gregg/Systems):**
   - Apply **RED-S** (Rate, Errors, Duration, Saturation) to all suspect services. High usage without saturation is a symptom, not a cause.
   - Enforce **Logical USE**: Apply Utilization/Saturation/Errors specifically to logical resources (thread pools, connection pools, queue depths). Apply non-linear Little's Law math when saturation nears 100%.
3. **UDA 3. Trace Dissection (1st Principles/Gregg):**
   - Execute **Critical Path Time Division**.
   - Evaluate **Span Count** (Structural Emergence / N+1 query checks) BEFORE evaluating **Span Duration**.
   - **The Dark Matter Pivot**: If the uninstrumented void (Trace Duration minus sum of Span Durations) > 20%, pivot immediately to host/infrastructure metrics (CPU schedule delay, GC pauses, network limits).
4. **UDA 4. Propagation & Origin (Systems/RCA):**
   - Differentiate errors that are **Generated** (originating locally) vs. **Propagated** (surfaced by a downstream dependency).
   - **Input Contract Rule**: Slow execution must be proven to not be the result of a sudden upstream payload/load change (e.g., larger batch size) before declaring the node itself faulty.
5. **UDA 5. Causal Triangulation (RCA):**
   - Anomalies must survive Dual Isolation: **Spatial Proof** (isolating the failing node) AND **Temporal Proof** (proving exact chronologic alignment with the macroscopic symptom).
   - **The Deductive Void**: Missing expected signals actively refute hypotheses. If a signal *should* be there logically but isn't, use that structural absence as evidence.

---

## Investigation Workflow (Steps 0–8)

### Step 0: Establish Time Context

If not in Session State: call `datetime_get_current_time`.

```
Current UTC time: [ts]
Investigation window: [start] – [end]   (default: last 15 min)
Duration: [X min/h]
```

For past-incident queries: set window around incident time ±15 min.

### Step 0.5: Fast-Path Triage

**MANDATORY: Check Tier 0 signals before querying any backend.**

1. `list_datasources()` → discover all available backends. **Parallel with any Tier 0 call.**
   Emit Signal Landscape: `Signal landscape: metrics=[uid|none] | traces=[uid|none] | logs=[uid|none]`
   Store in Session State. This governs which discovery paths are available in Steps 2–7.
   **Environment calibration — sense before proceeding:**
   - Any tier `none`? → mark it 🔲 UNAVAILABLE in Signal Coverage immediately. Skip its steps.
   - Any unfamiliar datasource type (not prometheus/loki/tempo/cloudwatch/clickhouse/mimir)?
     → call `get_query_examples(DatasourceType=<type>)` BEFORE querying it.
     **Unknown backend grade rule:** All findings from this backend default to `WEAK` until cross-validated by a finding from a known backend. State: `[UNKNOWN BACKEND: graded WEAK pending cross-check]`
   - All tiers `none` or all types unrecognizable? → trigger Autonomy Rule pause (state what IS available).
   - Only 1 signal type available? → state constraint explicitly: "Investigation scope limited to
     [type] — [missing types] not available in this environment."
2. **Environment Handshake:** Before deep architectural queries, execute a **volume baseline probe** on ALL available datastores for the target timeframe:
   - Metrics: `query_prometheus(expr="up{}", queryType="instant")` — any result = ONLINE
   - Logs: `query_loki_stats(logql="{}")` — entries > 0 = ONLINE
   - Traces: `tempo_traceql-metrics-instant(query="count_over_time({} [5m])")` — result > 0 = ONLINE
   If any store returns 0 results for baseline traffic → mark `[OFFLINE]`. Query ONLY online datastores in Step 4.
   - *Override Protocol:* If the User Constraint yields 0 anomalous results in the Initial Handshake, assume the user provided the wrong timestamp or service. Auto-expand the time window 4x and strip the service filter.
3. **Parallel calls (both safe):**
   - `list_alert_rules(datasourceUid=..., limit=1000)` → Filter by `state="firing"` (client-side)
   - `get_annotations(From=<start_ms>, To=<end_ms>)` → deployment markers, maintenance windows
3. **Alert quality check:** Weigh recently-transitioned `firing` alerts higher than long-standing or high-frequency alerts. Check `for` duration — a `1m` alert fires on noise, a `15m` alert fires on sustained issues.
4. Note severity impression: `Severity: [LOW/MEDIUM/HIGH/CRITICAL]` (from alert state + blast radius). Store in Session State.
   **Annotation candidate shortcut:** If an annotation (deploy/config/restart) has timestamp < symptom onset → add to Session State: `annotation_candidate: {type: "deploy", time: "<ts>", label: "<text>"}`. In Step 4.5, inject into `Timing & Spatial Proof`: "Annotation `<label>` at `<ts>` precedes symptom by ΔT." This seeds — but does not conclude — temporal causality.

5. **Decision:**
   - Firing alerts found? → Correlate with symptoms. Use alert labels as service discovery seed. If alerts fully explain symptom → Step 8 (TRIAGE).
   - No alerts? → Check annotations; proceed to Step 1.
   - **Known instrumentation gaps in Session State?** If `instrumentation_gaps` is non-empty AND the gap's signal tier is now available in Signal Landscape → probe the gap first in Step 4 and note: `[GAP RECHECK: previously absent, now instrumented]`. If still absent → confirm gap persists, skip.

### Step 1: Interpret & Hypotheses

- Restate user issue in 1 sentence
- **Extract Primary Constraint:** Identify immutable bounds in the user prompt (e.g., `user_id=123`, `time=14:00 UTC`). Add this to the Session State `Primary Constraint:` field. If the prompt is generic ("system is broken"), set to `[None provided]`.
- **Motivated Reasoning check:** Does the user's phrasing imply a preferred conclusion?
  Signal phrases: "confirm that...", "I think it's...", "shouldn't it be...", "just check X".
  If YES → emit before forming hypotheses: `⚠️ PRIOR DETECTED: [prior]. Suspended. Analysis below treats it as one hypothesis among peers — will confirm OR refute.`
  You MUST treat the user's prior as a peer hypothesis to be confirmed or refuted.
  **Budget triage position:** form as peer (equal epistemic standing), rank LAST in Step 3 triage ordering — test after alternative hypotheses. This maximizes anti-anchoring: agent sees counter-evidence first.
- **`<missing_context_gating>`:** If service name is not yet known (Step 2 not complete), mark ALL hypotheses formed here as `[ASSUMED-SERVICE]`. Commit to an investigation sequence ONLY AFTER Step 2 confirms the target service. Hypotheses are placeholders, not plans.
- **Systems context:** What upstream/downstream dependencies does this service have? What recently changed in this part of the system? (Check annotations, deploy history)
- Apply Known Failure Pattern fast-path (see below) — if matched, shortcut to indicated tier
- **Null Hypothesis Start:** Formulate hypotheses ONLY AFTER completing the initial Triage volume queries in Step 4.
- **Instrumentation hypothesis (always form):** "Is this environment's signal coverage sufficient to answer the question?" If Signal Landscape has any `none` or UNAVAILABLE tier, add to Hypothesis Tracker: "Root cause may reside in a signal type not exposed by this environment [INSTRUMENTATION_GAP risk]." This is not defeatism — it pre-arms the investigation against silent blind spots.
  *Auto-resolve rule:* If Signal Coverage shows ✅ (data found) for ALL tiers present in the Signal Landscape AND all tiers produce non-empty, non-void results for the target service → auto-REFUTE this hypothesis and note: `"Instrumentation hypothesis REFUTED: all available tiers returned data for service [X]."`
- State chosen investigation sequence AFTER service confirmed: "Starting with Metrics (latency issue). If inconclusive → Traces."
- **Ask yourself:** "If my first hypothesis is wrong, what would the evidence look like?" — this shapes what to query.
- **`--history` active?** Load history for service and seed hypotheses. If service unknown pre-Step 2, defer: set `history_pending=true`. @see library/history.md
- **Dependency signal check:** If ANY present → set `dependency_probe=true` in Session State: (1) user mentions upstream/downstream/cascading/dependency or names multiple services, (2) cross-service alerts firing in Step 0.5, (3) --history shows past multi-service causes, (4) dashboards reference multiple services, (5) multi-service deploys in annotations. Add hypothesis: "upstream dependency may have caused symptom in [target service]."
  **Topology shortcut (if `dependency_probe=true`):** Check historical dependencies from grade entries. @see library/topology.md. Store results in Session State `known_dependencies`.
- **Set Mode in Session State (INVESTIGATE paths):**
  TRIAGE → firing alert fully explains symptom → exit to Step 8 directly (0 analytical queries).
  DEEP DIVE → ≥3 backends expected, multi-service scope, or contradicting hypotheses from the start.
  STANDARD → all other cases (default). **Upgrade to DEEP DIVE mid-investigation if scope expands.**

### Step 2: Service Discovery (Once — Reuse Everywhere)

If service name in Session State: skip. If user provides exact names: use directly.

**Discovery order (try each available backend; use first successful result):**
1. `search_dashboards(query="<service_name>")` — find monitoring context
2. Tempo: `tempo_get-attribute-values(name="resource.service.name")` — OTel standard. Empty? → `tempo_get-attribute-names` to find the actual service identity attribute.
3. Metrics: try conventional labels (`job`, `service`, `app`) via `list_prometheus_label_values`. Empty? → `list_prometheus_label_names` to discover the service-identity label, then query its values.
4. Logs: try conventional labels (`service_name`, `app`, `container_name`) via `list_loki_label_values`. Empty? → `list_loki_label_names` to discover the service-identity label.

If `search_dashboards` returns matches: retrieve `get_dashboard_summary` + `get_dashboard_panel_queries` to understand instrumentation.

**No specific service named?** ("everything is slow", "errors across the board") — Dynamic Discovery:
1. **Tempo first** (if `traces` in Signal Landscape): query service graph metrics → `tempo_query_metrics(query="traces_service_graph_request_failed_total")` → reveals all services + error/latency profile in 1 query. Returns ranked candidate list.
2. **Prometheus fallback** (if Tempo unavailable or empty): `list_prometheus_metric_names(regex="<symptom_keyword>")` → discover affected metric domains → query `up{} == 0` + top 5 error rates by service-identity label.
3. **Logs fallback** (if Prometheus also empty): `search_loki(query="<symptom_keyword>", limit=20)` → group error logs by service label.
4. Cross-correlate candidates across available sources → produce ranked list:
   ```
   Service candidates: [service-A | confidence=HIGH (3/3 backends)] [service-B | MED (2/3)]
   ```
5. Pick highest-confidence candidate → state reasoning → proceed with standard discovery.

**Output + store in Session State:**

```
### SERVICE DISCOVERY
User provided: "[term]"
Service mapping:
  traces: <discovered_attr> = "[exact]"
  logs:   <discovered_label> = "[exact]"
  metrics: <discovered_label> = "[exact]"
Discovery method: [conventional | discovered via label_names]
```

**Name mismatch across backends?** → See Cross-System Service Correlation below.

**If `dependency_probe=true`:** If Session State `known_dependencies` already populated via Topology Shortcut (Step 1) → skip Dependency Discovery Cascade. Otherwise apply Dependency Discovery Cascade (Appendix D). Store results in Session State as `known_dependencies`.

### Step 3: Determine Investigation Sequence

- User provided Low Entropy Anchor (Trace ID)? → Anchor Fast-Path. Bypass discovery, extract trace. MUST pivot to metrics post-extraction to satisfy ADR-021 Proportionality.
- Alerts found (Step 0.5)? → Use alert labels as seed; begin at cheapest unqueried tier
- Known failure pattern matched? → Use pattern fast-path tier
- Otherwise → Apply signal cost hierarchy (Alerts done → Metrics → Traces → Logs)

**If `known_dependencies` populated:** Consider querying upstream service metrics before deeper tiers on target — confirming upstream failure is cheaper than diagnosing downstream symptoms. State: "Checking [upstream] first because [trigger signal]."

**Hypothesis count gate:** If ≥5 ACTIVE hypotheses at Step 3 entry → REFUTE ≥2 lowest-confidence hypotheses before querying. State: "Pruning H[N] (weakest evidence base): [1-line rationale]."
**Exception:** A PRIOR DETECTED hypothesis is immune to pruning — it can be ranked last (P-12) but cannot be eliminated before being tested.

**If ≥4 hypotheses ACTIVE:** Triage before querying — rank by: (1) highest evidence strength in favour, (2) cheapest signal tier to check, (3) highest blast radius if confirmed. State the ranking before proceeding. Do **not** distribute budget equally across all hypotheses.

**State sequence before querying. This step is MANDATORY.**

**CRITICAL:** Before querying each backend, complete Query Plan.

### Step 4: Query Backend

#### Common Pattern (All Backends)

1. Resolve datasource UID [from Session State or `list_datasources`]
2. Discover labels (label_names → label_values) — skip if already in Session State from Step 2
3. Complete Query Plan — classify intent, select function, document plan
4. Apply query language checklist (Appendix A/B/C) — rewrite non-compliant patterns
5. **Lag-Aware Triage:** When executing initial discovery queries, force a `[T0 - 15m, T0 + 5m]` sliding window. Query exact minute ranges ONLY AFTER isolating a specific spike, as pipeline ingestion latency will cause false negatives.
6. Execute query with correct type (Instant for values, Range for trends)
7. **Fidelity Check:** Before declaring an absence of data, check for telemetry sampling or rate-limiting warnings in output (if available). If fidelity is suspected to be degraded, mark findings as `[LOW-FIDELITY: Missing data possible]`.
8. **`<tool_persistence_rules>`:** A first empty result MUST trigger persistence protocols. Retry with:
   (a) alternate label/metric/attribute name, (b) broader time range (2×) — **run volume estimate first** (`query_loki_stats` for logs, `count_over_time` for metrics) before expanding; if volume is low, expand range rather than label, (c) label discovery fallback.
   (d) **Void Proof:** If persistent 0 results, query backend health metrics (e.g., Loki stats, Tempo up status). If pipeline is unhealthy/dropping data, mark `[AMBIGUOUS VOID]` instead of NO ANOMALY.
   (e) **5-Signal Checkpoint:** After every 5 analytical queries without a ROOT CAUSE, emit:
   `[CHECKPOINT] State=<S0-S4> | Budget=<N/ceil> | Lead=<H#, grade> | Gap=<what next query changes> | Query=<backend: type>`
   This forces DFA state awareness and hypothesis focus before the next query.
   ONLY after 2+ strategies exhausted → mark `[INSTRUMENTATION_GAP]` in Signal Coverage + continue.
9. Analyze: trend, spike, anomaly
10. Extract 1–5 key findings → **update Hypothesis Tracker + Signal Coverage**
    Each finding MUST include inline source tag: `[src: <tool_name> expr/query="..." → <key value>]`
    Example: `Error rate 4.7% [src: query_prometheus expr="rate(http_errors[5m])" → 0.047]`
    UNGROUNDED = finding emitted without src tag. Pre_output_verification Gate 2 checks for src tags.

#### Backend-Specific Notes

**Prometheus:** Start with `up{<service_label>="<svc>"}` (use label from Session State service mapping) to confirm service exists. Empty? → convention-first discovery: try `job`, `service`, `app` via `list_prometheus_label_values`. Step ≥ scrape interval (start with 60s). Use `query_prometheus_histogram` for percentiles. **Baseline:** For key metrics, compare across two horizons:
- **Trend** (is it worsening?): `offset 5m` → `offset 15m` → `offset 45m` — shows direction within the incident
- **Seasonal baseline** (is this abnormal?): `offset 7d` / `offset 14d` — same day-of-week comparison. Avoid `offset 1d` (weekend/weekday seasonality misleads).

**Loki:** `query_loki_stats` MANDATORY before broad pulls (>1M entries → narrow first). Start limit=10, expand cautiously. Direction: "backward" (newest first) for recent events.

**Tempo:** Identify search vs metrics query type. Latency: `duration > 500ms`. Errors: `status = error`. For large datasets: use metrics aggregations instead of search.

#### After Each Backend

- Extract 1–5 key findings with Evidence Strength grade. Each finding must carry inline source tag: `[src: tool expr/query="..." → key value]`
- **Critical check:** Does this evidence actually *explain* the symptom, or does it just *correlate*? What's the strongest counter-argument to the leading hypothesis?
- Update/confirm/refute hypotheses. **If all hypotheses are ACTIVE after 2+ queries — your hypotheses may be wrong. Step back and form new ones.**
- If 3 consecutive empty results → re-examine service name (wrong label? wrong time window?)
- **Temporal Bias check:** Are you treating a snapshot as a steady state? Fires when:
  (a) concluding "healthy" from a window <1h during a known intermittent issue;
  (b) using service mapping or topology from a previous session without re-verifying;
  (c) assuming infra config (scrape interval, sampling rate, retention) unchanged.
  If any applies → note inline: `[TEMPORAL ASSUMPTION: <X> treated as current — unverified]`
- If anomaly found but causal validation (Step 4.5) returns SYMPTOM → **pivot upstream:**
  1. If `known_dependencies` in Session State → pivot to most likely upstream (skip re-discovery)
  2. Otherwise → apply Dependency Discovery Cascade (Appendix D)
  3. Also check: log error messages for failing service/host names (e.g., "connection refused to auth-service:8080")
  Store discovered deps in Session State. Async/infrequent deps may not appear.
  → Pivot investigation to most likely upstream service
- **Root cause found → Step 4.5 → Step 8. Inconclusive → Step 5.**

### Step 4.5: Causal Reasoning Protocol

**MANDATORY before declaring root cause. Prevents symptom-as-cause errors.**

For each detected anomaly:

1. **Constraint Intersection Check** (MANDATORY): `[REVERSAL TEST] - What is the strongest possible argument that this finding is a RED HERRING unrelated to the Primary Constraint? If the argument is empirically sound, STATUS: RED HERRING. If the anomaly is a deployment/config change >5m ago but logically triggers the symptom, STATUS: LATENT_PRECONDITION. If the argument relies on assumptions, MATCH.` Discard `RED HERRING` findings and return to source hypothesis.
2. **Temporal Math (Bounded)**: Calculate explicit `ΔT` (Symptom Time - Anomaly Time). Evaluate against system tolerance bands: `Synchronous: ΔT < 1m`, `Async/Queue: ΔT < 15m`, `Batch: ΔT < 24h`. If `ΔT` violates the band AND is not a `LATENT_PRECONDITION`, status is `COINCIDENCE`.
3. **Common Cause Check (C Hidden Node)**: Before asserting `A caused B`, proactively disprove `C caused both A and B`. Check shared infrastructure (network layer, hypervisor, database cluster, identity provider). If shared infra degraded simultaneously, C is the root cause.
4. **Saturation vs Error Distinction**: You MUST NOT declare resource spikes (CPU/Memory/Network/Connections) as root causes unless you explicitly locate saturation evidence (throttling logs, OOMKilled events, connection pool exhaustion errors). High usage without saturation is a symptom, not a cause.
5. **Breadcrumb Anchors (Structural Keys)**: Hunt for IPs, unique correlation IDs, or framework classes. If a Structural Key is found *inside* a valid constrained log, pivot laterally to map the involved architecture.
6. **Cause or Symptom?** — If I fix X, does the original symptom disappear?
7. **Coincidence check** — Did X start BEFORE the symptom? Is the magnitude proportional?
   *Proportional: anomaly in X is directionally consistent AND ≥30% of the symptom’s relative magnitude vs pre-incident baseline. State both numbers explicitly.*
   *(Evidence Sufficiency: ROOT CAUSE declaration REQUIRES causal/saturation evidence beyond just identifying a dependency.)*
8. **One level deeper (2× max)** — "Why did X happen?" If the answer points to another system, THAT is the root cause candidate. Apply at most twice — if causal chain still leads outward after 2 levels → declare `[SYSTEMIC ROOT CAUSE: N layers]` naming all layers. Recurse MAXIMUM 2 levels. **Increment `Causal depth` in Session State after each application. At depth=2 → emit `[CAUSAL DEPTH: MAX]` and proceed to verdict immediately.**
   **`<dig_deeper_nudge>`:** Before item 9: ask what second-order cause produces the same signal? If a plausible alternative mechanism exists → add it to Hypothesis Tracker (ACTIVE) and return to Step 5 instead of proceeding to verdict. This gate fires pre-verdict — not after committing.
   **Causal depth increment (mandatory):** The CAUSAL VALIDATION block MUST include field `Causal depth: N → N+1` when this item fires. A CAUSAL VALIDATION block missing this field is INVALID when One-level-deeper was applied.
9. **Epistemic State Check** (Universal): Acknowledge Diagnostic Gaps (KU) as a broken causal chain requiring immediate traversal. Actively flag The Void (UU) when expected signals vanish across boundaries (e.g. Rate-Limiting Drop), recognizing this structural absence as the necessary deductive bridge to the final root cause.
10. **Feedback Loop Detection**: If Request Rate and Duration/Errors spike concurrently, explicitly hypothesize a feedback loop (e.g., "Retry Storm" or "Thundering Herd"). Look for retries from upstream or backoff failures.

```
## CAUSAL VALIDATION
Anomaly: [description]
├─ Timing & Spatial Proof: [Proof of alignment with macroscopic symptom]
├─ Reversal test: strongest evidence AGAINST this verdict = [___]
│   Absent → "Reversal clear — ¬ROOT CAUSE requires [X] which is absent — confirmed absent by: [specific query/tool call that returned empty or contradicting result]." You MUST name the query.
│   Present → downgrade to CONTRIBUTING FACTOR; do NOT declare ROOT CAUSE.
│   FORBIDDEN: "no strong counter-argument found" without naming a query that checked for it.
├─ Black boxes: [what was NOT checked that could invalidate this verdict — name ≥1]
│   Examples: unsampled traces, external deps absent from service graph, config-only changes
│   without metrics exposure, async/batch paths not in the investigation window.
├─ Tripartite Causality:
│   ├─ Trigger: [The active force, e.g., Traffic Spike, Deploy, Config Change]
│   ├─ Vulnerability: [The structural flaw, e.g., Unbounded memory pool, lack of backoff]
│   └─ Precondition: [Historical catalyst, if applicable, e.g., Data growth over months]
└─ Verdict: [ROOT CAUSE | CONTRIBUTING FACTOR | SYMPTOM | COINCIDENCE]
```

**Output requirement:** If verdict is ROOT CAUSE → output the CAUSAL VALIDATION block verbatim before proceeding to Step 8. The Step 8 root cause section MUST reference a completed CAUSAL VALIDATION block — no root cause claim is valid without one.

### Step 5: Continue or Stop?

**Drift check (RCoT):** Before evaluating the completion contract, reconstruct the user's original question from your current leading hypothesis. `ALIGNED` = hypothesis answers what was asked. `EXPANDED` = useful but broader — note it. `DRIFTED` = conclusion no longer addresses original question → reanchor before proceeding.

**Epistemic State (mandatory ONLY when path = KEEP GOING):**
Emit: `[EPISTEMIC: KK=<bound symptom — what is factually established> | KU=<causal gap to fill> | UU=<uninstrumented void if any> | next_query=<backend: function: label-filter>]`
The `next_query` field MUST specify backend + function + label selector/attribute filter. "Check logs" is FORBIDDEN — `Loki: query_loki_logs: {service="X"} |= "connection refused"` is required. Do NOT emit this block when path = DONE.

<completion_contract>
DONE WHEN:      All active hypotheses CONFIRMED or REFUTED, AND Signal Coverage shows
                ✅ or 🔲 for every signal (no ⬜ remains) relevant to active hypotheses.
BUDGET FINAL
OVERRIDE:       If `Budget regime = FINAL` → proceed to Step 8 immediately.
                This overrides ALL KEEP GOING conditions. State: `[BUDGET FINAL: synthesizing from current evidence]`
KEEP GOING IF:  Budget regime ≠ FINAL, AND ≥1 hypothesis is ACTIVE AND a specific query
                would change its status — name the backend + query before proceeding.
                AND within budget ceiling for active mode (STANDARD ≤8 / DEEP DIVE ≤15;
                EXPLORE/VALIDATE ≤5). Budget ceiling reached AND Δ-Quality >0 → emit
                budget-extension note and continue (see Operating Constraints). Hard ceiling 25.
                AND last query had non-zero Δ-Quality. Δ-Quality = ZERO when ALL hypothesis
                states are unchanged AND no new hypothesis seeded by most-recent query findings
                AND no root cause first identified. Hypothesis formed from prior evidence only
                (not from the most recent query) does NOT reset the Δ-Quality counter.
                Two consecutive Δ-Quality ZERO queries → 🛑 HALT and transition to `S4_RESOLUTION (BLOCKED)`.
                Mid-investigation hypothesis gate: If ≥5 ACTIVE hypotheses at this evaluation
                → REFUTE ≥1 lowest-confidence hypothesis before querying. State rationale.
BLOCKED FORMAT: "[BLOCKED: {signal}] — missing: {what}. Strategies tried: {specific query that failed}."
               Required additions to BLOCKED output:
               ├─ Owner signal: "Signal owned by: [infra|app|platform|unknown] team based on [datasource type]"
               ├─ Cheapest next step: "Fastest unblocking: [add <exporter> | check <dashboard> | ask <team> for <query>]"
               └─ Partial verdict: "Based on available evidence: [1-sentence best-effort conclusion or 'Insufficient data for any verdict']"
                Use after 2+ recovery strategies fail on an instrumentation gap, or Δ-Quality hits 0 twice.
                When marked BLOCKED: add [signal, backend] to Session State instrumentation_gaps.
NOTE: [INSTRUMENTATION_GAP] = Signal Coverage row marker (Step 4 tracking).
      [BLOCKED] = completion-level output tag (this contract).
</completion_contract>

**Instrumentation gap exception:** If a backend returned empty AND discovery attempts suggest it may not be instrumented for this service → mark as `[INSTRUMENTATION_GAP]` in Hypothesis Tracker AND Signal Coverage, treat as `[INSTRUMENTATION_GAP]` rather than a "no anomaly" backend.

### Steps 6–7: Query Backend 2 / Backend 3

Repeat Step 4 for next backend. **If a root cause candidate emerges from this backend → run Step 4.5 Causal Reasoning Protocol before proceeding to Step 8. CAUSAL VALIDATION is REQUIRED on secondary backends.**

**Cross-validation gate:** Before querying Backend 2/3, check: does any prior Backend finding contradict the current lead hypothesis? If yes AND unresolved → upgrade to CONFLICT format immediately (do NOT query a third backend to paper over the contradiction).

```
### CONTEXT FROM [BACKEND 1]
[1-sentence finding]
Now checking [BACKEND 2]. Service appears as: [label=value]
```

### Step 8: Synthesize & Output (Adaptive Depth)



**The Self-Healing Clause:** For incidents that show a clear return to baseline within the investigation window, the output MUST identify the exact time of recovery and systematically correlate it with any resolving actions found in annotations (e.g., pod restarts, reverts, auto-scaling).

| Findings | Output Mode |
|----------|-------------|
| Alert directly explained symptom (alert evidence self-sufficient — no analytical backend queries needed) | **TRIAGE** |
| 1–2 backends queried, clear root cause | **STANDARD** |
| 3+ backends queried, OR contradictions, OR dependency chain | **DEEP DIVE**. Required: ≥2 distinct backends queried before S4_RESOLUTION (unless Signal Landscape has ≤1 available backend). |
| No anomalies found across queried backends | **NO ANOMALY** |
| Backends return contradictory evidence about same event | **CONFLICT** |



<step_8_validation_gates>
TRIAGE:     Dimensional match required — alert must reference SAME symptom dimension as user report. Mismatch → escalate to STANDARD.
STANDARD:   Root cause section MUST embed CAUSAL VALIDATION block verbatim (not by reference).
DEEP DIVE:  ≥2 distinct backends queried before S4_RESOLUTION (unless Signal Landscape has ≤1). Multi-service: prefix "⛓️ [Target] ← upstream: [Root]".
NO ANOMALY: FORBIDDEN if Signal Coverage shows ⬜ for any available backend. INSTRUMENTATION_GAP variant when 🔲 ≥50%.
CONFLICT:   Selection rule: metric≠trace → cardinality; timestamp misalign → clock skew; label scope → scope; sampling < 100% → rate.
</step_8_validation_gates>

**Format template:** BEFORE constructing Step 8 output, load @see library/output.md — read ONLY the section matching the selected output mode.

**RULE:** Destructive actions REQUIRE explicit human confirmation via --ask-user.

**`--grade` active?** After outputting Step 8, invoke the grade protocol (@see library/grade.md): self-assess from conversation context, ask user for outcome verdict, append quality block to `resolutions/<service>.md`.

**Provisional grade capture:** Emit at the END of Step 8 output (before --grade Phase 2 user prompt):
  `[PROVISIONAL GRADE: outcome=<CONFIRMED_RCA|PARTIAL_RCA|NO_RCA> evidence=<grade> queries=<N> service=<service>]`
  This block can be manually appended to `resolutions/<svc>.md` if conversation terminates before Phase 2.
  When --grade Phase 2 runs, it supersedes the provisional block.

---

## Cross-System Service Correlation

| Signal Type | Identifier | Example |
|-------------|------------|---------|
| Traces | resource.service.name | "my-service" |
| Logs | Kubernetes labels | "my-service-default" |
| Metrics | job / service / deployment | "my-service-prod" |

**Resolution order** (try until match found):
1. Exact match across all backends
2. Strip namespace prefix (`default/`, `production/`)
3. Strip suffix (`-default`, `-prod`, `-svc`)
4. Case-insensitive retry
5. Fuzzy: `label_values` containing user term as substring

---

## Known Failure Pattern Fast-Paths

**These use conventional metric names.** Empty fast-path result ≠ pattern does not apply — metric name may differ. Discover via `list_prometheus_metric_names(regex="<keyword>")` before concluding pattern inapplicable or service not instrumented.

**Signal Landscape gate:** Validate the fast-path tier against Signal Landscape before routing. If fast-path tier is 🔲 unavailable → use cheapest available tier instead. Document: "Fast-path tier unavailable — routing to [alt tier]."

| Pattern | Fast-Path Tier | Primary Signal |
|---------|----------------|----------------|
| "crash", "restart", "OOMKilled" | Metrics | `container_restarts` + memory, then Logs for last error |
| "connection refused", "pool exhausted" | Metrics | `connections_used` vs max, then Logs for pool errors |
| "deployment", "just deployed" | Metrics | error_rate delta before/after deployment timestamp |
| "disk", "storage", "no space left" | Metrics | `node_filesystem_avail_bytes` |
| "5xx", "error rate", "requests failing" | Metrics | `http_requests` total/errors, then Traces for error spans |
| "slow", "latency", "high response time" | Metrics | `histogram_quantile` p99/p95, then Traces for slow traces |
| "slow" / "unresponsive" with **no explicit error signals** | Metrics | Resource saturation first: CPU (`node_cpu_seconds_total`), memory (`container_memory_working_set_bytes`), connections (`connection_pool_active`) vs baseline — latency is the symptom, saturation is the candidate cause |

---

*Worked example: @see context.md (Quick-Reference Example section).*

---

## Rationalization Counters

| Anti-Pattern | Rule |
|---|---|
| Query before label discovery | **FORBIDDEN** — discovery IS the speed optimization; no labels = no valid query |
| Parallel analytical queries | **FORBIDDEN** — analytical queries are sequential; discovery-tier parallelism only (see Parallelism Policy) |
| Empty result = absence | **FORBIDDEN** — empty may mean wrong label/name; exhaust `<tool_persistence_rules>` before concluding gap |
| Skip Tier 0 (alerts/annotations) | **FORBIDDEN** — cheapest tier is mandatory; empty alert list is itself evidence |
| Root cause without CAUSAL VALIDATION | **FORBIDDEN** — no root cause claim valid without a completed CAUSAL VALIDATION block |

*Specific query-language anti-patterns (PromQL/LogQL/TraceQL) → see Appendices A–C.*

---

## Tool Failure Handling

When MCP tools fail (timeout, datasource unreachable, unexpected error):

1. **Retry once** with same parameters
2. **If persistent:** Document failure in Session State, note which signal type is unavailable
3. **Adapt:** Continue investigation with remaining backends. Mention gap in Step 8.
4. Base conclusions ONLY on successful tool outputs. State: "Unable to query [signal type]: [error]"

---

**Query language rules:** Apply @see library/query-rules.md checklist for the target backend before constructing each analytical query.

**Dependency discovery:** When `dependency_probe=true`, apply @see library/deps.md cascade.
