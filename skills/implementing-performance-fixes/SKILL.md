---
name: implementing-performance-fixes
description: "[PHASE 4] Implements and validates fixes. Entry: PERFORMANCE_THEORIES.md has theory with 'Deeper Investigation: SKIP' OR theory with status 'verified'. Outputs: Code fix, benchmarks run, BENCHMARK_BASELINE.md updated, theory marked fixed/falsified/fix-failed. Next: Loop to PHASE 2 if performance gaps remain."
---

# Implementing Performance Fixes

Implements performance fixes and validates them with benchmarks. Handles both theories that skipped verification (definitive evidence) and theories that went through deep investigation.

## Core Principle

**Benchmarks are the ultimate validation. Profiling suggests, benchmarks prove.**

## Prerequisites

- **REQUIRED**: `PERFORMANCE_THEORIES.md` with theory having:
  - `Status: pending` + `Deeper Investigation: SKIP`, OR
  - `Status: verified` (from Phase 3)
- **REQUIRED**: `BENCHMARK_BASELINE.md` with benchmark descriptions and execution commands

## Quick Start

**Step 1**: Load theory to implement from `PERFORMANCE_THEORIES.md`
- Select highest-priority theory with:
  - `Status: pending` + `Deeper Investigation: SKIP`, OR
  - `Status: verified`

**Step 2**: Design fix approach (see [WORKFLOW.md](WORKFLOW.md) Phase 2)

**Step 3**: Implement the fix

**Step 4**: Run 3 relevant benchmarks to validate

**Step 5**: Based on results:
- **Improved** → Keep fix, run ALL benchmarks, update BENCHMARK_BASELINE.md + theory status
- **Degraded/No change** → Revert fix, update theory status, optionally run additional profiling to diagnose why

## Workflow Overview

```text
1. Load verified theory with evidence
2. Design fix (architectural changes allowed if cost-benefit fits)
3. Implement fix
4. Run 3 relevant benchmarks
5a. If improved → run all benchmarks → update BENCHMARK_BASELINE → mark theory "fixed"
5b. If degraded → revert → mark theory "fix-failed"
```

## Benchmark Selection

Select 3 benchmarks from BENCHMARK_BASELINE.md based on their documented focus:

1. Read each benchmark's **Rationale** field
2. Match rationale to the code path affected by your fix
3. Prioritize benchmarks that exercise the fixed code

## File Updates

### BENCHMARK_BASELINE.md

Add new column to results table:

```markdown
| Benchmark | Baseline | After Fix #N |
|-----------|----------|--------------|
| queens    | 32ms     | 24ms         |
```

### PERFORMANCE_THEORIES.md

Update theory status:

```markdown
**Status**: fixed  <!-- was: verified -->
**Fix Applied**: [Date] - [Brief description of fix]
**Performance Impact**: [X% improvement on benchmark Y]
```

Or if fix failed:

```markdown
**Status**: fix-failed  <!-- was: verified -->
**Fix Attempted**: [Date] - [What was tried]
**Reason**: [Why it didn't improve performance]
```

## Status Values

| Status | Meaning |
|--------|---------|
| `pending` | Theory generated, awaiting implementation or verification |
| `verified` | Theory proven by Phase 3 deep investigation |
| `fixed` | Fix applied and benchmarks show improvement |
| `falsified` | Theory disproven (either in Phase 3 or after implementation failed) |
| `fix-failed` | Fix attempted but reverted due to regression/no improvement |

## Entry Paths

**Path 1: Skip Verification** (for definitive evidence + low-risk)
- Theory has `Deeper Investigation: SKIP` and `Status: pending`
- Implement directly based on Phase 2 evidence
- If benchmarks fail → Mark `falsified` (theory was wrong despite apparent evidence)

**Path 2: After Verification** (for theories requiring deep investigation)
- Theory has `Status: verified` (from Phase 3)
- Implement based on verified findings
- If benchmarks fail → Mark `fix-failed` (theory was correct but fix approach was wrong)

## Integration

**Predecessor**: `deep-performance-investigation` → Provides verified theories

**Successor**: Loop back to `broad-performance-investigation` if more issues exist

## Workflow Position

```text
1. [establishing-benchmark-baseline] → Create baseline (PHASE 1)
2. [broad-performance-investigation] → Generate theories (PHASE 2)
3. [deep-performance-investigation]  → Verify with tools (PHASE 3)
4. [implementing-performance-fixes]  → THIS SKILL (PHASE 4)
5. Loop to step 2 if performance gaps remain
```

See [WORKFLOW.md](WORKFLOW.md) for detailed phase instructions.
