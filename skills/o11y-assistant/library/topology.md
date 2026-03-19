# --topology Protocol

Invoke as a standalone flag: `--topology <service>`

**Purpose:** Extract a structured service topology map from accumulated `--grade` YAML entries for the given service. Answers: which services appear upstream/downstream, how often each is causal, and what the most common Trigger/Vulnerability/Precondition patterns are.

## Prerequisites

- `resolutions/<service>.md` must exist and contain ≥1 grade entry with YAML schema block (Phase 4 of `--grade`).
- If file is missing or has no YAML entries: emit `[TOPOLOGY: insufficient history — run investigations with --grade first]` and stop.

## Step 1: Parse Grade Entries

Read all YAML `# grade-entry` blocks from `resolutions/<service>.md`.
Extract per-entry:
- `upstream_causes` → list of services that appeared as causal upstream
- `downstream_affected` → list of services affected downstream
- `tripartite.trigger`, `tripartite.vulnerability`, `tripartite.precondition`
- `root_cause` text
- `user_verdict`

## Step 2: Build Topology Map

```
## TOPOLOGY MAP — [service] — [N entries analyzed] — [UTC timestamp]

### Upstream Services (callers / causes)
| Service | Appearances | Times Causal | Confidence |
|---------|-------------|--------------|------------|
| svc-A   | 4/8         | 3/8          | HIGH       |

### Downstream Services (callees / affected)
| Service | Appearances | Times Affected | Confidence |
|---------|-------------|----------------|------------|
| db-main | 6/8         | 5/8            | HIGH       |

### Most Common Failure Patterns
| Trigger            | Count | Example root cause |
|--------------------|-------|--------------------|
| Traffic Spike      | 3     | "connection pool..."  |
| Deploy             | 2     | "error rate after..." |

### Most Common Vulnerabilities
| Vulnerability      | Count |
|--------------------|-------|
| Unbounded pool     | 3     |
| Missing backoff    | 2     |

### Blind Spot Register (cumulative)
- [blind spot text] — appeared in N/M investigations
```

## Step 3: Output

Emit the TOPOLOGY MAP in the response.
Offer: "Run `--review <service>` to compact the prose entries, or start a new investigation with `--history` to use this topology as a hypothesis seed."

## Notes

- Confidence = HIGH if service appears in >50% of entries, MEDIUM 25–50%, LOW <25%.
- Services with `user_verdict: wrong` entries are excluded from causal counts.
- This flag does NOT query any observability backend. It is purely historical analysis.
