---
name: implementing-performance-fixes
description: Implements fixes for verified performance theories in Truffle language implementations. Analyzes verified issues, designs fixes (including architectural changes when cost-benefit fits), validates improvements with benchmarks, and updates tracking files. Use after verifying a theory to apply and validate the fix. Use when fixing performance issues, implementing optimizations, or validating that changes improve performance.
---

# Implementing Performance Fixes

Transforms verified performance theories into working fixes with measured validation.

## Core Principle

**Verified theories identify the problem. This skill implements and validates the solution.**

## Prerequisites

- **REQUIRED**: `PERFORMANCE_THEORIES.md` with at least one theory status `verified`
- **REQUIRED**: `BENCHMARK_BASELINE.md` with benchmark descriptions and execution commands
- **REQUIRED**: Theory verification evidence from `verifying-performance-theories`

## Quick Start

**Step 1**: Load the most critical verified theory from `PERFORMANCE_THEORIES.md`

**Step 2**: Design fix approach (see [WORKFLOW.md](WORKFLOW.md) Phase 2)

**Step 3**: Implement the fix

**Step 4**: Run 3 relevant benchmarks to validate (select based on benchmark focus in BENCHMARK_BASELINE.md)

**Step 5**: Based on results:

- **Improved** → Keep fix, run all benchmarks, update files
- **Degraded** → Revert fix, update theory status

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
| `verified` | Theory proven, awaiting fix |
| `fixed` | Fix applied and validated |
| `fix-failed` | Fix attempted but reverted |
| `falsified` | Theory disproven during verification |

## Integration

**Predecessor**: `verifying-performance-theories` → Provides verified theories

**Successor**: Loop back to `generating-performance-theories` if more issues exist

## Workflow Position

```text
1. [establishing-benchmark-baseline] → Create baseline
2. [generating-performance-theories] → Generate theories
3. [verifying-performance-theories]  → Verify with tools
4. [implementing-performance-fixes]  → THIS SKILL
5. Loop to step 2 if performance gaps remain
```

See [WORKFLOW.md](WORKFLOW.md) for detailed phase instructions.
