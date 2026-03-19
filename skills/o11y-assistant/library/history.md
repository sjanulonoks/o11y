# --history Protocol

## Loading

Read `resolutions/<service>.md`. Resolve `<service>` from prompt context
or Session State. If service unknown (pre-Step 2), defer: set
`history_pending=true` in Session State. After Step 2 resolves the service
name, load history before proceeding to Step 3.

**If file doesn't exist:**
> Emit: `[HISTORY: No past investigations for <service>. Investigation #1 — baseline mode. No historical seeds available.]`
> Set `History: none_available` in Session State. Skip all steps below. Do NOT form a contradicting hypothesis (nothing to contradict).

## Staleness Check (requires H-01 metadata header in Distilled Patterns)

Before reading Distilled Patterns, parse the metadata header:
```
## Distilled Patterns — last: <UTC> — covered: <N> entries — total: <M>
```
Compute `new_entries = M - N`.

- If `new_entries ≥ 5` → emit `[HISTORY: Distilled Patterns stale by N entries — auto-triggering --review first]` and invoke --review protocol before continuing.
- If Distilled Patterns section missing AND total raw entries ≥ 5 → same trigger.
- If `new_entries < 5` OR total < 5 → proceed directly.

## Interpretation

- If file has `## Distilled Patterns` section with Weighted Pattern Ranking table (v0.57+):
  use table directly — more reliable than raw entries. Assess confidence:
  - **CONFIDENCE: HIGH** if top pattern has `Confirmed ≥ 3` AND `Invalidated = 0` AND `Last Seen < 30 days`
  - **CONFIDENCE: LOW** if top pattern has `Confirmed ≤ 2` OR `Invalidated > 0` OR `Last Seen > 90 days`
  - Emit at Step 1: `[HISTORY CONFIDENCE: HIGH|LOW] — top pattern: [Root Cause] (Confirmed N, Invalidated M)`
- If file has legacy prose `## Distilled Patterns` (pre-v0.57): use patterns directly.
- If file has only raw entries: scan most recent 5 entries.
  Extract: root causes, blind spots, user corrections.

## Integration into Step 1

- Add top-ranked pattern's `Root Cause` as Step 1 hypothesis
- **Structured Contrast Protocol** (replaces free-form "MUST form ≥1 contradicting hypothesis"):
  From top-ranked pattern, extract `Trigger` and `Vulnerability`. Form two grounded contrasts:
  - `[HISTORICAL CONTRAST A]`: `"Trigger was NOT [historical Trigger] — probe for alternative trigger [e.g., config change, dependency failure]"`
  - `[HISTORICAL CONTRAST B]`: `"Vulnerability is NOT [historical Vulnerability] — what other structural flaw could produce the same symptom?"`
  Both added to Hypothesis Tracker at Step 1. Skip this step if CONFIDENCE: LOW or `History: none_available`.
- Past blind spots → hypothesis seeds ("last time we missed X — check it")
- Past `user_verdict: wrong` corrections → **highest-priority** hypothesis seeds (agent was wrong here before — don't repeat). These override CONFIDENCE level.
- Past PARTIAL_RCA / NO_RCA outcomes → signal to try different backends or query strategies

## Anti-Anchoring Rule

History informs — it never shortcuts discovery. Do NOT skip service discovery
or label validation because history says "it's always the connection pool."
