# Dependency Discovery Cascade

> Auto-discover service dependencies. Try each tier in order — **stop at first that produces results.**

**Naming:** Upstream = calls us (we are `server`). Downstream = we call them (we are `client`).

---

## Tier 1: Service Graph Metrics & Spanmetrics (Mapping Macroscopic Symptoms)

1. **Service Graph** (1–2 queries):
   ```
   list_prometheus_metric_names(regex="traces_service_graph_request_total")
   → if exists:
     Downstream: query_prometheus('sum by (client, connection_type) (rate(traces_service_graph_request_total{server="<svc>"}[5m])) > 0')
     Upstream:   query_prometheus('sum by (server, connection_type) (rate(traces_service_graph_request_total{client="<svc>"}[5m])) > 0')
   ```
2. **Spanmetrics** (1–3 queries):
   ```
   list_prometheus_metric_names(regex="span.*|traces_spanmetrics.*")
   → if exists:
     list_prometheus_label_names(matches="<spanmetric>{...}")
     list_prometheus_label_values(labelName="<found_label>", matches="<metric>{<service_label>=\"<svc>\"}")
   ```

---

## Tier 2: TraceQL Dynamic Discovery (Resolving Known-Unknowns)

**The Epistemic Tension:** Use ONLY if Tier 1 metrics confirm a macroscopic symptom but fail to isolate the specific causal node due to cardinality compression. The agent must acknowledge the mathematical limit of the metric before proceeding.

**1. Dynamic Sensing Loop:**
```
tempo_get-attribute-names
```
🔴 **FORBIDDEN:** Hardcoding target attributes (like `peer.service`) is a premature RCA conclusion. You MUST dynamically derive available attributes from reality.

**2. The Spatial Proof (Instant Aggregation):**
Construct dynamic queries using the sensed attributes to structurally isolate the fault (e.g., finding the peer generating the highest counts).
```
tempo_traceql-metrics-instant(query='topk(5, {status=error} | count_over_time(...))')
```
🔴 **FORBIDDEN:** LEAVE `most_recent=false` (the default) to avoid severe performance penalty.
🟢 **ALWAYS USE:** `with(sample=true)` — Explicitly rely on Tempo's statistical guarantee of the failure shape for spatial isolation.

**3. The Temporal Proof (Causal Range Validation):**
Discovering a fault node spatially does not prove causality (it could be background noise). You MUST satisfy the Step 4.5 Coincidence Check by calculating its derivative against the macroscopic symptom timeline.
```
tempo_traceql-metrics-range(query='rate({status=error && peer.service="<discovered_peer>" | count_over_time(...)}[1m])')
```

---

## Output

```
Store in Session State:
  known_dependencies:
    upstream: [svc-a (sync), svc-b (messaging_system)]
    downstream: [svc-c (sync), db-main (database)]
    source: service_graph | spanmetrics | traceql_discovery
```
