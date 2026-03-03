# --history <service> Detailed Protocol

## Loading

Read `resolutions/<service>.md`. Resolve `<service>` from prompt context
or Session State. If service unknown (pre-Step 2), defer: set
`history_pending=true` in Session State. After Step 2 resolves the service
name, load history before proceeding to Step 3.

## Interpretation

- If file has `## Distilled Patterns` section (written by --review):
  use patterns directly as hypothesis seeds. More reliable than raw entries.
- If file has only raw entries: scan most recent 5 entries.
  Extract: root causes, blind spots, user corrections.

## Integration into Step 1

- Add most recent historical root cause as a hypothesis
- MUST form >=1 hypothesis that contradicts historical pattern
  ("what if it's NOT [past cause]?")
- Past blind spots -> hypothesis seeds ("last time we missed X — check it")
- Past user corrections -> highest-priority hypothesis seeds
  (agent was wrong here before — don't repeat)
- Past PARTIAL_RCA / NO_RCA outcomes -> signal to try different backends
  or different query strategies than last time

## Anti-Anchoring Rule

History informs — it never shortcuts discovery. Do NOT skip service discovery
or label validation because history says "it's always the connection pool."
