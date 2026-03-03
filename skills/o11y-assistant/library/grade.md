# --grade Protocol

Invoke after Step 8 in the same conversation. Do NOT invoke during investigation.

## Phase 1: Agent Self-Assessment

Review the conversation and produce:

```
## INVESTIGATION QUALITY — [service] — [UTC timestamp]
- **Root cause:** [1 sentence + evidence grade]
- **Key finding:** [non-obvious insight, or "none"]
- **Blind spot:** [what wasn't checked and why]
- **Outcome:** [CONFIRMED_RCA | PARTIAL_RCA | NO_RCA | UNCERTAIN]
- **Efficiency:** [N] queries, [M] dead ends, sequence [optimal|suboptimal: why]
- **Compliance:** causal validation [done|skipped], label discovery [done|skipped],
  query plans [all|partial|none]
```

## Phase 2: User Verdict

Ask:
> Was the root cause correct? (yes / partially / no / too early to tell)
> If partially or no: what was the actual cause? (1 sentence)

Overwrite the `Outcome` line with user's answer. If user provides correction,
append: `- **Correction:** [user's actual cause]`

## Phase 3: Record

Append full block to `resolutions/<service>.md`.
Resolve `<service>` from Session State.
Create `resolutions/` directory and file if first time.
Check existing entries for novelty — skip if this is a duplicate root cause
AND no new blind spots or findings.
