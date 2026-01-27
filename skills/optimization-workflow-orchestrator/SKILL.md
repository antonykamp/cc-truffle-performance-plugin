---
name: optimization-workflow-orchestrator
description: "[ORCHESTRATOR] Manages performance optimization workflow for Truffle languages. Entry: User wants to optimize performance. Process: Determines current state, executes appropriate phase skill (1→2→3/4→4), loops until performance goals met. Phase 3 is conditional based on 'Deeper Investigation' field. IMPORTANT: Single entry point for all optimization work."
---

# Optimization Workflow Orchestrator

Orchestrates the complete performance optimization workflow. Single entry point that manages state, transitions, and termination.

## Quick Start

**Step 1**: Determine current state (see State Detection below)

**Step 2**: Execute the appropriate phase skill

**Step 3**: After phase completes, re-evaluate state and continue or terminate

## State Detection

Check files in this order to determine current phase:

```text
1. BENCHMARK_BASELINE.md exists?
   NO  → Execute PHASE 1 (establishing-benchmark-baseline)
   YES → Continue to check 2

2. PERFORMANCE_THEORIES.md exists?
   NO  → Execute PHASE 2 (broad-performance-investigation)
   YES → Continue to check 3

3. Any theory with status "verified"?
   YES → Execute PHASE 4 (implementing-performance-fixes) for verified theory
   NO  → Continue to check 4

4. Any theory with status "pending" + "Deeper Investigation: SKIP"?
   YES → Execute PHASE 4 (implementing-performance-fixes) for this theory
   NO  → Continue to check 5

5. Any theory with status "pending" + "Deeper Investigation: REQUIRED"?
   YES → Execute PHASE 3 (deep-performance-investigation) for this theory
   NO  → Continue to check 6

6. All theories have status "fixed" or "fix-failed" or "falsified"?
   YES → Evaluate termination (see below)
   NO  → Error state, ask user
```

## Phase Skills

| Phase | Skill | Purpose |
|-------|-------|---------|
| 1 | `establishing-benchmark-baseline` | Create benchmarks and baseline |
| 2 | `broad-performance-investigation` | Analyze code, generate theories |
| 3 | `deep-performance-investigation` | Prove/disprove theories with tools |
| 4 | `implementing-performance-fixes` | Implement fix, validate improvement |

## Termination Criteria

After all theories are processed (fixed/fix-failed/falsified), evaluate:

**Continue optimization (loop to PHASE 2)** if:

- Performance still below baseline expectations (check BENCHMARK_BASELINE.md)
- User requests more optimization
- Significant performance gaps remain (>20% slower than expected)

**Stop optimization** if:

- Performance meets or exceeds baseline expectations
- User indicates satisfaction
- No more theories can be generated (diminishing returns)
- All recent fixes failed (indicates deeper issues)

## Edge Cases

| Situation | Action |
|-----------|--------|
| All theories falsified | Loop to PHASE 2 with different focus |
| No performance gap found | Optimization complete, congratulate user |

## Workflow Visualization

```text
                    ┌──────────────────────────────────────┐
                    │                                      │
                    ▼                                      │
┌─────────┐    ┌─────────┐                     ┌─────────┐│
│ PHASE 1 │───▶│ PHASE 2 │────────────────────▶│ PHASE 4 │┘
│Baseline │    │Theories │                     │  Fix    │
└─────────┘    └─────────┘                     └─────────┘
                    │                                ▲
                    │ (if Deeper Investigation       │
                    │  = REQUIRED)                   │
                    ▼                                │
               ┌─────────┐                           │
               │ PHASE 3 │───────────────────────────┘
               │ Verify  │
               └─────────┘
                    ▲                                │
                    │         (if gaps remain)       │
                    └────────────────────────────────┘
                                   │
                                   ▼ (if done)
                              ┌─────────┐
                              │  DONE   │
                              └─────────┘

Phase 2 → Phase 4 directly if "Deeper Investigation: SKIP"
Phase 2 → Phase 3 → Phase 4 if "Deeper Investigation: REQUIRED"
```

## Orchestrator Checklist

Copy this to track progress:

```text
Optimization Progress:
- [ ] State determined
- [ ] PHASE 1: Baseline established (BENCHMARK_BASELINE.md exists)
- [ ] PHASE 2: Theories generated (PERFORMANCE_THEORIES.md exists)
- [ ] PHASE 3: Theory verified (at least one verified)
- [ ] PHASE 4: Fix implemented and validated
- [ ] Termination evaluated
- [ ] Loop or complete
```

## Integration

This skill CALLS the phase skills:

- `establishing-benchmark-baseline` (PHASE 1)
- `broad-performance-investigation` (PHASE 2)
- `deep-performance-investigation` (PHASE 3)
- `implementing-performance-fixes` (PHASE 4)

This skill does NOT call tool skills directly. Tool skills are called by PHASE 3.

