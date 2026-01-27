---
name: deep-performance-investigation
description: "[PHASE 3 - CONDITIONAL] Targeted profiling focused on specific theory. Entry: PERFORMANCE_THEORIES.md has theory with 'Deeper Investigation: REQUIRED'. Outputs: Theory marked verified/falsified. Next: PHASE 4 (implementing-performance-fixes). SKIP this phase if theory has 'Deeper Investigation: SKIP'."
---

# Deep Performance Investigation

Deep investigation of specific performance theories using targeted profiling tools. This phase is CONDITIONAL - only run for theories marked "Deeper Investigation: REQUIRED".

## Core Principles

- **Broad profiling (Phase 2) discovers issues. Targeted profiling (Phase 3) proves specific theories.**
- **Phase 2 used tools on entire benchmark. Phase 3 uses tools focused on ONE theory/function.**
- **Skip this phase if Phase 2 already provided definitive evidence for low-risk fixes.**

## Quick Start

**Entry Condition**: Theory in `PERFORMANCE_THEORIES.md` with `Deeper Investigation: REQUIRED`

**When to Use**:
- Theory has CIRCUMSTANTIAL evidence (general patterns without specific tool confirmation)
- OR theory requires HIGH-RISK fix (architectural changes, multi-file, expensive)

**When to SKIP**:
- Theory has `Deeper Investigation: SKIP` → Go directly to Phase 4 (implementing-performance-fixes)

**Approach**: Verify one theory at a time, highest-severity first, using targeted profiling focused on that specific theory.

**Output**: Theory marked `verified` or `falsified`

## Workflow Overview

1. **Select** highest-priority unverified theory
2. **Prepare** - load tool docs, form expectations (Fermi estimation)
3. **Execute** verification - use profiling tools OR direct implementation (if fix is <10 lines)
4. **Handle emergent issues** - pivot if more critical, note as future work if not
5. **Synthesize** evidence → VERIFIED / FALSIFIED / INCONCLUSIVE
6. **Next steps** - if verified, recommend fix and stop; otherwise continue

See [WORKFLOW.md](WORKFLOW.md) for detailed procedures.

## Tool Skills

Use the following tool skills for targeted profiling:

| Purpose | Tool Skill |
|---------|-----------|
| Hot function identification | `profiling-with-cpu-sampler` |
| Execution frequency | `tracing-execution-counts` |
| Optimization barriers | `detecting-performance-warnings` |
| Compilation behavior | `tracing-compilation-events` |
| Inlining analysis | `tracing-inlining-decisions` |
| Type stability | `detecting-deoptimizations` |
| Allocation patterns | `profiling-memory-allocations` |
| Deep IR analysis | `analyzing-compiler-graphs` |

**Note on Compiler Graphs**: When theories come from code analysis (e.g., "this allocation should be eliminated"), compiler graphs provide **direct evidence** of what the compiler actually did. Use them early for allocation/boxing theories, not as a last resort.

## Common Pitfalls

- ❌ **Running only one tool** - Multiple tools required for confidence
- ❌ **Skipping Fermi verification** - Silent tool failures produce garbage data
- ❌ **Ignoring emergent issues** - If tools reveal a critical issue (like deopt loops), evaluate whether to pivot
- ❌ **Always pivoting** - Only pivot if the new issue is MORE critical than the current theory

## Targeted vs Broad Profiling

**Phase 2 (Broad)**: Profiled entire benchmark to discover issues
- `--cpusampler` → All functions
- `--memtracer` → All allocations
- IGV dump → Overview of multiple functions

**Phase 3 (Targeted)**: Profile ONE specific theory with filters
- `--cpusampler` with focused benchmark run
- `--compiler.TracePerformanceWarnings=all --vm.Djdk.graal.MethodFilter='*specificFunction*'`
- IGV dump with `--engine.CompileOnly='*specificFunction*'`
- Deopt tracing with `--engine.CompileOnly=specificFunction`

Use tool skill documentation for proper filter syntax.

## Related Skills

**Predecessor**: `broad-performance-investigation` → Provides theories to verify

**Successor**: `implementing-performance-fixes` → Implements and validates fixes for verified theories

**Tool Skills Used**:
- `profiling-with-cpu-sampler`
- `tracing-execution-counts`
- `detecting-performance-warnings`
- `tracing-compilation-events`
- `tracing-inlining-decisions`
- `detecting-deoptimizations`
- `profiling-memory-allocations`
- `analyzing-compiler-graphs`

## Workflow Position

```text
1. [establishing-benchmark-baseline] → Create baseline (PHASE 1)
2. [broad-performance-investigation] → Generate theories (PHASE 2)
3. [deep-performance-investigation]  → THIS SKILL (PHASE 3)
4. [implementing-performance-fixes]  → Implement and validate fix (PHASE 4)
5. Loop to step 2 if performance gaps remain
```
