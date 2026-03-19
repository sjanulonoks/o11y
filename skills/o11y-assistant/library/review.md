# --review <service> Protocol

Standalone operation. No active investigation required.

## Step 1: Load

Read `resolutions/<service>.md`. If file doesn't exist: "No investigation
history for <service>."

## Step 2: Analyze

Count and categorize from all grade entries (YAML Phase 4 blocks where available, prose where not):

- **Total investigations** (M)
- **Entries since last distillation** (M minus `covered` from Distilled Patterns header, or M if no header)
- **Outcome distribution** (CONFIRMED_RCA / PARTIAL_RCA / NO_RCA / UNCERTAIN)
- **Recurring root causes** — group by similarity, count occurrences
- **Correction anti-weighting** — identify entries where `user_verdict: wrong`. For each:
  - Tag root cause as `Invalidated +1`
  - If `Invalidated / Confirmed ≥ 0.3` → mark pattern `[PARTIAL: N invalidations]`
- **Time-decay weighting** — for each root cause pattern, compute:
  ```
  Weight = Confirmed × RecencyFactor
  RecencyFactor: Last Seen < 30 days = 1.0 | 30–90 days = 0.7 | > 90 days = 0.4
  ```
  Sort patterns by Weight descending → this determines Rank
- **Recurring blind spots** (group by similarity)
- **Efficiency trend** (comparing avg queries across oldest 1/3 vs newest 1/3 of investigations)
- **Compliance gaps** (which steps most often skipped)
- **User corrections** (where was agent wrong, and what was the actual cause)

## Step 3: Distill

Produce to conversation:

```
## SERVICE REVIEW: [service] ([N] investigations)
Outcomes:            [distribution]
Efficiency trend:    [improving | stable | degrading], avg [N] queries
Compliance gaps:     [list or "none"]
User corrections:    [list or "none"]
Invalidated patterns: [root cause — N invalidations | "none"]

Top patterns (by weighted rank):
| Rank | Root Cause | Trigger | Vulnerability | Confirmed | Invalidated | Weight | Last Seen |
|------|-----------|---------|---------------|-----------|-------------|--------|-----------|
| 1    | ...        | ...     | ...           | N         | M           | W      | YYYY-MM-DD |

Recurring blind spots: [list or "none"]
```

## Step 4: Compact-Rewrite

**Pre-write validation (all must pass before writing):**
- [ ] Distilled Patterns table has ≥ 1 row
- [ ] Top pattern has `Confirmed ≥ 2` (skip this gate if total investigations < 5)
- [ ] Any `user_verdict: wrong` entries reflected in `Invalidated` counts
- [ ] New file length < original length
- If any gate fails → emit reason and do NOT write. Leave existing file untouched.

Rewrite `resolutions/<service>.md` with:

1. `## Distilled Patterns` section at top with metadata header and Weighted Pattern Ranking table:
   ```markdown
   ## Distilled Patterns — last: <UTC> — covered: <M> entries — total: <M>
   | Rank | Root Cause | Trigger | Vulnerability | Confirmed | Invalidated | Weight | Last Seen |
   |------|-----------|---------|---------------|-----------|-------------|--------|-----------|
   | 1    | ...        | ...     | ...           | N         | M           | W.W    | YYYY-MM-DD |
   ```
   Patterns marked `[PARTIAL: N invalidations]` are included but visually flagged.

2. Deduplicated raw entries below — merge investigations that share the same root
   cause AND same Trigger into a single entry, keeping:
   - first occurrence date, last occurrence date, count
   - union of blind spots
   - any user corrections
   - Tripartite (Trigger / Vulnerability / Precondition) from the best-graded entry

3. Unique investigations (different root cause OR different Trigger) preserved individually.

The resulting file MUST be shorter than the input. If it isn't, something
went wrong — don't write.
