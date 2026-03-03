# --review <service> Protocol

Standalone operation. No active investigation required.

## Step 1: Load

Read `resolutions/<service>.md`. If file doesn't exist: "No investigation
history for <service>."

## Step 2: Analyze

Count and categorize:
- Total investigations
- Outcome distribution (CONFIRMED_RCA / PARTIAL_RCA / NO_RCA / UNCERTAIN)
- Recurring root causes (group by similarity, count occurrences)
- Recurring blind spots (same)
- Efficiency trend (are investigations getting faster?)
- Compliance gaps (which steps are most often skipped?)
- User corrections (where was the agent wrong, and what was the actual cause?)

## Step 3: Distill

Produce to conversation:

```
## SERVICE REVIEW: [service] ([N] investigations)
Outcomes:            [distribution]
Top root causes:     [cause (Nx), cause (Nx), ...]
Recurring blind spots: [list]
Efficiency trend:    [improving | stable | degrading], avg [N] queries
Compliance gaps:     [list or "none"]
User corrections:    [list or "none"]

Distilled patterns:
- [pattern 1 — actionable observation]
- [pattern 2]
- [pattern 3 if warranted]
```

## Step 4: Compact-Rewrite

Rewrite `resolutions/<service>.md` with:

1. `## Distilled Patterns` section at top (from Step 3 output)
2. Deduplicated entries below — merge investigations that share the same root
   cause into a single entry, keeping: first occurrence date, last occurrence
   date, count, union of blind spots, any user corrections
3. Unique investigations preserved individually

The resulting file should be shorter than the input. If it isn't, something
went wrong — don't write.
