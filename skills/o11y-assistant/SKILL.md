---
name: o11y-assistant
version: 0.33
description: >
  ALWAYS USE when investigating incidents, checking system health, exploring services,
  validating hypotheses, or querying ANY observability backend (Prometheus/Mimir,
  Loki, Tempo, CloudWatch, ClickHouse, or any Grafana-connected datasource).
  Triggers on @rca or @o11y prefix, or mentions of latency, errors, crashes,
  degradation, SLO breach, error budget burn, or root cause analysis.
---

# O11y Assistant — Unified Observability

## Core Principle

Evidence-based investigation across **all available** observability signals. Single unified workflow. You ARE the expert — you call tools directly. You do NOT delegate.

**Backend Abstraction:** This workflow operates on **signal types**, not hardcoded backends. `list_datasources()` reveals what's available. Adapt accordingly.

| Signal Type | Typical Backends | Discovery |
|-------------|-----------------|-----------|
| ALERTS | Grafana alert rules (any datasource) | `list_alert_rules` |
| METRICS | Prometheus, Mimir, CloudWatch, InfluxDB | `list_datasources(type="prometheus")` etc. |
| TRACES | Tempo, Jaeger | `list_datasources(type="tempo")` etc. |
| LOGS | Loki, ClickHouse, CloudWatch Logs | `list_datasources(type="loki")` etc. |

**Unknown backend?** → `get_query_examples(DatasourceType=<type>)` to self-teach query syntax before proceeding.

> **@see** Load references ONLY when entering the relevant step:
> - Step 0.5 → `references/tools_grafana.md` (alerts, annotations)
> - Step 4 → `references/query_planning.md` + `references/tools_<backend>.md` + `references/investigation_recipes.md`
> - Handoff → `references/handoff_protocol.md` (only if emitting handoff)

---

## Intent Classification & Routing

**Before any tool call, classify user request into exactly one mode:**

| Mode | Triggers | Route |
|------|----------|-------|
| **INVESTIGATE** | Active/recent incident, degradation, errors | Full workflow (Steps 0–8) |
| **EXPLORE** | "What services/metrics exist?", health overview | Step 0 → 0.5 → Step 2 (discover services) → Query `up{} == 0` (down services) + top 5 error rates by job → Step 8 NO ANOMALY or TRIAGE |
| **VALIDATE** | User has hypothesis; confirm or deny | Step 0 → Step 2 → query relevant backend → confirm/deny → Step 8 |
| **DISCOVER** | "Show me available dashboards/alerts/services" | list_datasources + search_dashboards + list_alert_rules → structured catalogue |

**User expertise inference:**
- Vague symptom ("things are slow") → novice: explain each step briefly
- Exact service + time + symptom → expert: skip explanations, go directly to evidence

---

## Session State

**When any of the following are discovered, record and reuse — do NOT re-discover:**

```
SESSION STATE
─────────────────────────────────────────────────────
Datasource UIDs:   metrics=<uid>  logs=<uid>  traces=<uid>  profiles=<uid>
Service mapping:   <user_term> → traces:<name>, logs:<label>=<val>,
                                 metrics:job=<val>
Time context:      current_utc=<ts>  investigation_window=<start>–<end>
SLO context:       SLIs=[availability|latency|throughput|correctness]
                   error_budget_status=[healthy|warning|exhausted|unknown]
Severity:          [LOW|MEDIUM|HIGH|CRITICAL]
─────────────────────────────────────────────────────
```

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
- Deployment/infrastructure event → check annotations first
- Stack trace / log pattern mentioned → start Tier 4
- Slow dependency / distributed flow → start Tier 3
- CPU/memory hotspot mentioned → start Tier 3.5 (profiles)
- User names specific backend → honor it

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
| Query language discipline | Apply cost rules (@see references/) to every query. |

### Parallelism Policy

```
✅ SAFE to parallelize:
   - datetime_get_current_time + list_datasources
   - list_alert_rules + get_annotations (both Tier 0)
   - list_alert_rules + label discovery (when service known)

❌ NEVER parallelize:
   - Any two analytical queries (Tier 2+)
   - Queries to different backends simultaneously
```

---

## Evidence Strength Grading

Assign to every finding before using it to support a conclusion:

| Grade | Criteria | Use in Conclusions |
|-------|----------|--------------------|
| **STRONG** | Direct metric/trace/log showing causal mechanism | Can support root cause claim |
| **MODERATE** | Temporal correlation + plausible mechanism | Can support hypothesis, needs corroboration |
| **WEAK** | Temporal correlation only | Flag as observation, not evidence |
| **SPECULATIVE** | Pattern match without direct evidence | Mention in "areas for further investigation" only |

---

## Hypothesis Tracking

Maintain this table across all investigation steps. Update after EVERY backend query:

```
## HYPOTHESIS TRACKER
| # | Hypothesis | Evidence For | Evidence Against | Strength | Status |
|---|-----------|-------------|-----------------|----------|--------|
| 1 | [statement] | [findings] | [findings] | STRONG/MOD/WEAK | ACTIVE/CONFIRMED/REFUTED |
| 2 | ... | ... | ... | ... | ... |
```

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

### Step 0.25: SLO Context (If Available)

Before investigating symptoms, establish:
- Which SLIs does this symptom potentially affect? (availability, latency, throughput, correctness)
- What is the current error budget burn rate? (Query SLO metrics if available, or ask user)
- Is the error budget exhausted? → Set severity to HIGH/CRITICAL
- **WHY:** An SLO violation reframes the investigation from "find root cause" to "restore SLO compliance" — different stopping conditions and urgency.

If SLO data unavailable: set `error_budget_status=unknown`, continue. Do NOT block on this.

### Step 0.5: Fast-Path Triage + Severity Classification

**MANDATORY: Check Tier 0 signals before querying any backend.**

**@see** `references/tools_grafana.md` for tool parameters.

1. `list_datasources()` → alert-capable datasource UID [skip if in Session State]
2. **Parallel calls (both safe):**
   - `list_alert_rules(datasourceUid=..., limit=1000)` → Filter by `state="firing"` (client-side)
   - `get_annotations(From=<start_ms>, To=<end_ms>)` → deployment markers, maintenance windows
3. **Auto-classify severity** from triage signals:

```
## SEVERITY SCORE (sum dimensions, 0–3 each)

D1 Alert Severity:   0=none | 1=warning | 2=critical/single-svc | 3=critical/multi-svc
D2 Error Budget:     0=healthy(>50%) | 1=warning(10–50%) | 2=critical(<10%) | 3=exhausted
                     Detect: slo:error_budget_remaining:ratio | if unavailable: default 1
D3 Blast Radius:     0=leaf service (no dependents)
                     1=service with ≤3 downstream dependents
                     2=service with >3 dependents OR multiple independent services
                     3=infrastructure (DNS/ingress/DB/queue/mesh) OR confirmed cascade
                     Detect: traces (upstream spans), dashboard names, service name
                     heuristics (db|redis|kafka|rabbitmq|nginx|envoy|istio → 3)
D4 Duration:         0=<5min | 1=5–15min | 2=15–60min | 3=>60min
D5 User Impact:      0=internal/batch only | 1=internal users (admin/ops)
                     2=external users (degraded) | 3=revenue path (payments/checkout/auth)
                     Detect: service name (payment|checkout|auth|signup → 3,
                     admin|batch|cron|worker → 0), alert labels (critical_path=true → 3)
                     If uncertain: default 1

Total (0–15) → Severity:  0–4=LOW | 5–8=MEDIUM | 9–12=HIGH | 13–15=CRITICAL
```

   **Burn rate shortcut:** If multi-window burn rate alerts exist (Google SRE): fast-burn 14.4x/1h → CRITICAL, slow-burn 3x/3d → MEDIUM.

#### Classification Confidence

```
confidence = (dimensions_with_actual_data / 5) × 100%
  ≥80% (4–5/5 from real data) → HIGH confidence
  60–79% (3/5)                 → MEDIUM confidence
  <60% (≤2/5)                  → LOW confidence
```

If LOW confidence:
- Announce: `⚠️ Severity [X] at LOW confidence ([N]/5 dimensions from data)`
- Use NEXT HIGHER severity's investigation budget

   Store severity + confidence in Session State. Re-evaluate at Post-Query Checkpoint (Step 4.2).

4. **Decision:**
   - Firing alerts found? → Correlate with symptoms. Use alert labels as service discovery seed. If alerts fully explain symptom → Step 8 (TRIAGE).
   - No alerts? → Check annotations; proceed to Step 1 if no context.

### Step 1: Interpret & Hypotheses

- Restate user issue in 1 sentence
- Apply Known Failure Pattern fast-path (see below) — if matched, shortcut to indicated tier
- Otherwise form 1–2 architecture-aware hypotheses → **add to Hypothesis Tracker**
- State chosen investigation sequence: "Starting with Metrics (latency issue). If inconclusive → Traces."

### Step 2: Service Discovery (Once — Reuse Everywhere)

If service name in Session State: skip. If user provides exact names: use directly.

**Discovery order (try each available backend; use first successful result):**
1. `search_dashboards(query="<service_name>")` — find monitoring context
2. Tempo: `tempo_get-attribute-values(name="resource.service.name")` — OTel standard
3. Metrics: `list_prometheus_label_values(labelName="job")` — fallback
4. Logs: `list_loki_label_values(labelName="service_name")` — fallback

**Output + store in Session State:**
```
### SERVICE DISCOVERY
User provided: "[term]"
Service mapping:
  traces: resource.service.name = "[exact]"
  logs:   k8s_deployment_name = "[exact]" | service_name = "[exact]"
  metrics: job = "[exact]" | service = "[exact]"
```

### Step 3: Determine Investigation Sequence

- Alerts found (Step 0.5)? → Use alert labels as seed; begin at cheapest unqueried tier
- Known failure pattern matched? → Use pattern fast-path tier
- Otherwise → Apply signal cost hierarchy (Alerts done → Metrics → Traces → Logs)

**State sequence before querying. Do not skip this.**

### Step 4: Query Backend

**@see** `references/query_planning.md` — MANDATORY: Complete Query Plan before every analytical query.
**@see** `references/tools_<backend>.md` — Load the relevant backend reference before constructing queries.

#### Common Pattern (All Backends)

1. Resolve datasource UID [from Session State or `list_datasources`]
2. Discover labels (label_names → label_values)
3. Complete Query Plan (classify intent, select function, document plan)
4. Apply query language cost rules (from reference file) — rewrite non-compliant patterns
5. Execute query with correct type (Instant for values, Range for trends)
6. Handle empty/NaN results (empty = no data, not always "no problem")
7. Analyze: trend, spike, anomaly
8. Extract 1–5 key findings → **update Hypothesis Tracker**

**After each backend → run POST-QUERY CHECKPOINT (Step 4.2).**

### Step 4.2: Post-Query Checkpoint

**Run this checklist in order after EVERY backend query. Stop at first triggered action.**

#### 1. Budget Check

Tier 0 queries (alerts/annotations/dashboards) are FREE. Count Tier 2+ only.

Budget: LOW=6 | MEDIUM=10 | HIGH=15 | CRITICAL=20

- At 75% → `⚠️ BUDGET: [N]/[M] queries. Narrowing focus.`
- At 100% → `🔴 BUDGET: Exhausted.` → Step 8. (CRITICAL can request +5 with justification)

#### 2. Evidence Update

- Assign Evidence Strength to each new finding
- Update Hypothesis Tracker: confirm, refute, or add hypotheses

#### 3. Drift Scan

Check all 7 triggers. Apply Graduated Response based on stage (Early 1-3 / Mid 4-7 / Late 8+ queries):

**Data-Level Triggers:**

| # | Trigger | Condition | Early | Mid | Late |
|---|---------|-----------|-------|-----|------|
| 1 | **EMPTY WELL** | 3 consecutive empty/NaN | Re-examine name/time | Try alternate labels | Accept absence as evidence |
| 2 | **SCOPE CREEP** | 3+ services beyond target | Suspicious → return | Check dependency graph | Likely cascading → document |
| 3 | **CONTRADICTION** | Findings contradict | Recheck queries | Check sampling/alignment | Contradiction IS the signal |
| 4 | **DIMINISHING RETURNS** | Last 2 queries redundant | Broaden: different metric family | Narrow: focus strongest hypothesis | STOP backend → Step 8 |

**Cognitive Bias Triggers:**

| # | Trigger | Condition | Action |
|---|---------|-----------|--------|
| 5 | **CONFIRMATION LOCK** | H1 ACTIVE 3+ steps, zero Evidence Against | Run one REFUTATION query for H1 |
| 6 | **ANCHOR FIXATION** | >50% queries target user's initial metric | Run one query outside anchor scope |
| 7 | **RECENCY OVERRIDE** | Conclusions favor last backend over stronger earlier evidence | Re-weight by Evidence Strength |

**Recovery cost:** Max 2 corrective analytical queries per trigger. Discovery queries (label_names/values) are FREE.

`⚠️ DRIFT: [trigger name]. [1-sentence action taken].`

#### 4. Compound Check (Overrides Individual Triggers)

| Compound | Triggers | Action |
|----------|----------|--------|
| **LOST INVESTIGATION** | EMPTY WELL + SCOPE CREEP | 🔴 FULL RESET to Step 0.5. Original triage may be wrong. |
| **EVIDENCE COLLAPSE** | CONTRADICTION + CONFIRMATION LOCK | 🔴 Present ALL evidence to user without interpretation. |
| **RESOURCE EXHAUSTION** | DIMINISHING RETURNS + Budget ≥75% | 🔴 Immediate Step 8. Do NOT spend remaining budget. |

#### 5. Severity Re-Evaluation

Re-score D1-D5. Then apply modifiers:

**Trend Modifier** (compare last 3 data points of primary metric):
- ACCELERATING (worsening) → severity +2
- STABLE (±10%) → no change
- RECOVERING (improving) → severity -1 (floor 0). Do NOT auto-close.

**Duration Escalation** (one-way, irreversible):
- Investigation >30 min AND ≤ MEDIUM → upgrade to HIGH
- Investigation >60 min AND ≤ HIGH → upgrade to CRITICAL
- CRITICAL + >90 min → recommend human escalation

If severity changed → log transition:
```
SEVERITY HISTORY: [T+Nm] [OLD] → [NEW] (reason)
```

#### 6. Decision

Root cause found (with evidence) → Step 4.5 → Step 8. Inconclusive → Step 5.

### Step 4.3: Investigation Memory

After each completed investigation (Step 8), append ONE lesson to conversation context:

```
MEMORY: [service] [trigger] → [lesson]
```

**Scope:** Memory lives in the current conversation. It does NOT persist across separate conversations.

**Check at two points:**
1. **Before Step 2:** "Have I investigated this service earlier in this conversation?"
2. **When a drift trigger fires:** "Is this a pattern I've already encountered?"

### Step 4.5: Causal Reasoning Protocol

**MANDATORY before declaring root cause. Prevents symptom-as-cause errors.**

For each detected anomaly, answer:

1. **Cause or Symptom?** — If I fix X, does the original symptom disappear?
2. **Coincidence check** — Did X start BEFORE the symptom? Is the magnitude proportional?
3. **One level deeper** — "Why did X happen?" If the answer points to another system, THAT is the root cause candidate.

```
## CAUSAL VALIDATION
Anomaly: [description]
├─ Cause or symptom? [cause | symptom of ___]
├─ Timing: X started [before|after|simultaneously] with symptom
├─ One level deeper: X happened because [___]
└─ Verdict: [ROOT CAUSE | CONTRIBUTING FACTOR | SYMPTOM | COINCIDENCE]
```

### Step 5: Continue or Stop?

**Stopping rules (severity-conditional):**

| Severity | Condition | Action |
|----------|-----------|--------|
| Any | Evidence supports root cause (passed Step 4.5) | → Step 8 |
| LOW/MEDIUM | 2+ backends queried, no anomaly | STOP → Step 8 ("No anomaly detected") |
| HIGH/CRITICAL | 2+ backends queried, no anomaly | **ESCALATE** — don't stop. Document what was ruled out. Flag for human investigation. |
| Any | Further queries would be speculative | STOP → Step 8 |

**Continue only if:** Current backend inconclusive AND evidence points to specific next backend. State which and exactly why.

### Steps 6–7: Query Backend 2 / Backend 3

Repeat Step 4 for next backend.

```
### CONTEXT FROM [BACKEND 1]
[1-sentence finding]
Now checking [BACKEND 2]. Service appears as: [label=value]
```

### Step 8: Synthesize & Output (Adaptive Depth)

| Findings | Output Mode |
|----------|-------------|
| Alert directly explained symptom (≤2 tool calls) | **TRIAGE** |
| 1–2 backends queried, clear root cause | **STANDARD** |
| 3+ backends queried, complex multi-cause incident | **DEEP DIVE** |
| No anomalies found across queried backends | **NO ANOMALY** |

#### All Modes Include:

- **What We Ruled Out:** Hypotheses tested and refuted (prevents re-investigation)
- **Severity History:** Transition log (STANDARD/DEEP DIVE only)

#### TRIAGE Format
```
🔴 [Service]: [what happened in 1 sentence]
Root cause: [1 sentence, evidence-backed, with Evidence Strength grade]
Immediate action: [1–2 bullets]
Ruled out: [hypotheses refuted]
```

#### STANDARD Format
1. **Summary** — What / Impacted / Window
2. **Timeline** (chronological reconstruction from all evidence)
   ```
   [T-15m] Normal baseline (Metrics: error rate 0.1%)
   [T-5m]  Deployment detected (Annotation: deploy v2.3.1)
   [T-2m]  Error rate spike (Metrics: 0.1% → 4.7%)
   [T-0]   Alert fires (Grafana: "High Error Rate" → firing)
   ```
3. **Evidence** — Per-backend findings with Evidence Strength grades
4. **Root Cause** — Evidence-based with causal validation
5. **Actions**
   - 🔴 **Mitigate Now** — Stop the bleeding (rollback, restart, scale)
   - 🟡 **Remediate This Week** — Fix the specific failure
   - 🟢 **Prevent Long-Term** — Systemic improvement (alerts, architecture, testing)

#### DEEP DIVE Format
Standard format PLUS:
6. **Contributing Factors** — Beyond root cause: what conditions enabled the failure?
7. **Evidence Appendix** — Queries used, raw findings, deeplinks
8. **Prevention Recommendations** — Alerting gaps, recording rules, architectural observations

#### NO ANOMALY Format
```
✅ [Service]: No anomalies detected in [window]
Checked:
  Metrics: error rate [X%], p99 latency [Yms], saturation [Z%] — within baseline
  Traces:  [N] traces sampled, no error spans, p99 [Yms]
  Logs:    no error-level entries in window
Possible explanations:
  - Issue resolved before investigation (check recent deploy/restart)
  - Issue is intermittent → suggest: set alert on [specific metric]
  - Issue outside instrumented scope → suggest: check [uninstrumented area]
Ruled out: [hypotheses tested]
```

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

**If all fail after 3 attempts:** Ask user for exact service name. Document resolution in Session State.

---

## Known Failure Pattern Fast-Paths

| Pattern | Fast-Path Tier | Primary Signal |
|---------|----------------|----------------|
| "crash", "restart", "OOMKilled" | Metrics | `container_restarts` + memory, then Logs for last error |
| "connection refused", "pool exhausted" | Metrics | `connections_used` vs max, then Logs for pool errors |
| "deployment", "just deployed" | Metrics | error_rate delta before/after deployment timestamp |
| "disk", "storage", "no space left" | Metrics | `node_filesystem_avail_bytes` |
| "5xx", "error rate", "requests failing" | Metrics | `http_requests` total/errors, then Traces for error spans |
| "slow", "latency", "high response time" | Metrics | `histogram_quantile` p99/p95, then Traces for slow traces |

---

## Rationalization Counters

| Rationalization | Counter |
|-----------------|---------|
| "Query without label discovery first" | FORBIDDEN: Discovery IS the speed optimization |
| "Parallel backend queries save time" | FORBIDDEN for analytical queries. Discovery calls only. |
| "Backend 1 normal → issue elsewhere" | FORBIDDEN: Continue to Backend 2 only if justified |
| "Expand time window without volume check" | FORBIDDEN: Volume check first always |
| "Backend 3 will have signal if 1+2 don't" | FORBIDDEN: 2 backends no signal = STOP (unless HIGH severity) |
| "Calculate answer from intermediate data" | FORBIDDEN: Query directly for what user asked |
| "rate() × time is close enough for counts" | FORBIDDEN: Use count aggregations for exact counts |
| "Query planning obvious, skip documentation" | FORBIDDEN: Complete query plan before EVERY analytical query |
| "Instant and Range queries interchangeable" | FORBIDDEN: Match to user intent |
| "Skip alerts — go straight to metrics" | FORBIDDEN: Tier 0 is cheapest. Empty alert list = evidence. |
| "Don't search dashboards — query directly" | FORBIDDEN: Dashboards provide service discovery + instrumentation context |
| "Skip annotations — focus on symptom" | FORBIDDEN: Annotations correlate timing with events. MANDATORY. |
| "Tool timed out — give up on this backend" | FORBIDDEN: Retry once. If persistent, document failure and continue to next backend. |

---

## Tool Failure Handling

When MCP tools fail (timeout, datasource unreachable, unexpected error):

1. **Retry once** with same parameters
2. **If persistent:** Document the failure in Session State, note which signal type is unavailable
3. **Adapt:** Continue investigation with remaining backends. Mention the gap in Step 8 output.
4. **Never hallucinate data** from a failed tool call. State: "Unable to query [signal type]: [error]"

---

## Cross-Skill Handoff

**@see** `references/handoff_protocol.md` for the full orchestration-layer design (lifecycle state machine, parallel handoffs, resolution feedback).

### Skill Boundaries

**This skill owns:** Symptom detection, evidence gathering, root cause identification, severity classification, investigation output.

**This skill does NOT own:** Remediation execution, infrastructure provisioning, incident communication, post-incident documentation, capacity planning.

### When to Emit a Handoff

When investigation identifies a root cause requiring action outside observability, **complete Step 8 first**, then format recommended actions.

### Handoff Output Format

Append to Step 8 output:

```
## RECOMMENDED ACTIONS (Outside Observability Scope)
──────────────────────────────────────────────
target_domain:   [deployment | infrastructure | capacity | comms]
action_needed:   [rollback | restart | scale | notify | investigate_further]
severity:        [from auto-classification]
confidence:      [HIGH | MEDIUM | LOW]

Context:
  root_cause:    [1-sentence summary]
  service:       [service name]
  evidence:      [max 3 findings with strength grades]
  ruled_out:     [max 3 hypotheses refuted]
  deeplinks:     [Grafana explore links]
──────────────────────────────────────────────
```

### Handoff Patterns

| Pattern | When | What the Agent Does |
|---------|------|--------------------|
| **Explicit** (default) | No other skills | Present as "Recommended next steps (requires human action)" |
| **Suggested** | User mentions `@deploy` etc. | Suggest: "Run `@deploy rollback payment-service v2.3.0`" |

**RULE:** Never directly trigger destructive actions (rollbacks, deletions). Always require human confirmation.
