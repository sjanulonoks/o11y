# Cross-Skill Handoff — Orchestration Protocol Reference

> This document describes the **orchestration-layer design** for cross-skill handoffs.
> It is NOT loaded during normal investigations — the agent cannot execute these directly.
> It serves as a specification for building an orchestration layer around the o11y-assistant skill.

---

## Handoff Lifecycle State Machine

Track every handoff through these states. Emit transitions as structured events.

```
PROPOSED ──→ ACCEPTED ──→ IN_PROGRESS ──→ COMPLETED ──→ VERIFIED ✅
   │              │             │               │
   ├→ REJECTED    │             ├→ FAILED       └→ UNRESOLVED
   └→ EXPIRED     │             └→ ESCALATED         │
        │         │                  │                └→ REINVESTIGATE
        └─────────┴──────────────────┘
                      ↓
               HUMAN_ESCALATION
```

| State | Owner | Next States |
|-------|-------|-------------|
| **PROPOSED** | o11y | ACCEPTED, REJECTED, EXPIRED |
| **ACCEPTED** | target | IN_PROGRESS |
| **REJECTED** | target (scope mismatch) | → HUMAN_ESCALATION |
| **EXPIRED** | system (no response) | → HUMAN_ESCALATION |
| **IN_PROGRESS** | target | COMPLETED, FAILED, ESCALATED |
| **COMPLETED** | target | VERIFIED, UNRESOLVED |
| **FAILED** | target | → HUMAN_ESCALATION |
| **VERIFIED** | o11y (symptom resolved) | → [end] |
| **UNRESOLVED** | o11y (symptom persists) | → REINVESTIGATE |

**Expiry timeout (severity-scaled):** CRITICAL = 2 min, HIGH = 5 min, MEDIUM = 15 min, LOW = 30 min.

---

## Skill Discovery

Before emitting a handoff signal, check if a matching skill exists:

1. **File-based** (default): Check `[workspace]/.agents/skills/*/SKILL.md` for capability metadata in YAML frontmatter
2. **Convention-based** (zero-config): If user mentions `@deploy` → assume deploy-assistant exists
3. If found → Pattern B (Suggested). If not found → Pattern A (Explicit/human).

**Expected capability metadata in target skills:**
```
# In target skill's YAML frontmatter:
domains: [deployment, rollback, canary]
actions: [rollback, deploy, promote, abort]
accepts_handoff: true
escalates_to: [human | infra-assistant]
```

---

## Parallel Handoffs

When root cause requires actions across multiple domains:

1. Emit **separate handoff signals** per domain (each has independent lifecycle)
2. Tag all with shared `incident_id` so targets can coordinate
3. Track lifecycle per handoff independently
4. **Max 3 parallel handoffs.** If >3 actions needed → CRITICAL incident requiring human incident commander.

```
Example:
  HANDOFF #1: target=deployment, action=rollback,  incident_id=INC-2024-0115
  HANDOFF #2: target=infrastructure, action=scale,  incident_id=INC-2024-0115

Ordering (if needed):
  HANDOFF #2: depends_on=HANDOFF #1  (scale AFTER rollback)
```

---

## Post-Handoff Verification

```
Post-Handoff Check (after COMPLETED):
1. Wait [timeout based on severity]
2. Re-query the metric/alert that triggered investigation
3. Symptom resolved → VERIFIED: "✅ Remediation confirmed effective"
4. Symptom persists → UNRESOLVED: "⚠️ Remediation did not resolve. Re-investigating..."
```

---

## Resolution Feedback (Postmortem Loop)

After every VERIFIED or UNRESOLVED handoff, capture a resolution report:

```
RESOLUTION REPORT
──────────────────────────────────────────────
handoff_id:         [from handoff signal]
actual_root_cause:  [what remediation revealed]
diagnosis_accurate: [YES | PARTIAL | NO]
action_taken:       [what was actually done]
time_to_resolve:    [ACCEPTED → VERIFIED duration]
lessons:            [alert gaps, threshold changes, runbook needs]
──────────────────────────────────────────────
```

**Learning rules:**
- `diagnosis_accurate = NO` → Log discrepancy. Inform future Drift Detection.
- `diagnosis_accurate = YES` for 5+ same-pattern incidents → Candidate for automated runbook.
- `diagnosis_accurate = PARTIAL` → Root cause was deeper. Strengthen causal validation.
