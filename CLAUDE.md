# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code plugin for analyzing and optimizing GraalVM Truffle language performance. It provides a suite of specialized skills for profiling, tracing, and diagnosing performance issues in Truffle language implementations.

## Plugin Structure

Skills are organized in the `skills/` directory. Each skill is a self-contained directory with:
- `SKILL.md` - Main skill documentation with frontmatter metadata (name, description)
- Additional `.md` files - Supporting documentation (WORKFLOW.md, PATTERNS.md, QUERIES.md, etc.)

The plugin manifest is located at `.claude-plugin/plugin.json`.

## Performance Optimization Workflow

This plugin implements a systematic 4-phase performance optimization loop:

### Phase Architecture

**PHASE 1: Establishing Benchmark Baseline** (`establishing-benchmark-baseline`)
- Entry condition: No `BENCHMARK_BASELINE.md` exists
- Creates benchmark suite from Benchmarks Game and/or AreWeFastYet
- Outputs: Benchmark files + `BENCHMARK_BASELINE.md` with performance expectations
- Next: PHASE 2

**PHASE 2: Generating Performance Theories** (`broad-performance-investigation`)
- Entry condition: `BENCHMARK_BASELINE.md` exists AND (no `PERFORMANCE_THEORIES.md` OR all theories fixed)
- Runs broad profiling (cpusampler, memtracer, IGV overview) on entire benchmark
- Performs systematic code analysis for optimization opportunities
- Assesses evidence quality (DEFINITIVE vs CIRCUMSTANTIAL) and fix complexity (LOW-RISK vs HIGH-RISK)
- Makes "Deeper Investigation" decision: SKIP or REQUIRED
- Outputs: `PERFORMANCE_THEORIES.md` with theories containing evidence quality and investigation decision
- Next: PHASE 3 if any theory has "Deeper Investigation: REQUIRED", else PHASE 4

**PHASE 3: Verifying Performance Theories** (`deep-performance-investigation`) - CONDITIONAL
- Entry condition: Theory with "Deeper Investigation: REQUIRED" (circumstantial evidence OR high-risk fix)
- Runs targeted profiling with filters focused on specific theory/function
- Uses same tool suite as Phase 2 but with targeted focus (not broad discovery)
- Marks theories as verified/falsified
- Next: PHASE 4 when theory is verified
- **Skip this phase** if theory has "Deeper Investigation: SKIP" (definitive evidence + low-risk fix)

**PHASE 4: Implementing Performance Fixes** (`implementing-performance-fixes`)
- Entry condition: Theory with "Deeper Investigation: SKIP" OR theory with status "verified"
- Implements fix, runs 3 relevant benchmarks
- If improved: runs ALL benchmarks, updates `BENCHMARK_BASELINE.md`, marks theory "fixed"
- If failed: reverts, marks "falsified" (if skipped Phase 3) or "fix-failed" (if verified in Phase 3)
- Optionally runs additional profiling to diagnose why fix failed
- Next: Loop to PHASE 2 if performance gaps remain

### Orchestrator Skill

`optimization-workflow-orchestrator` - Single entry point that:
1. Detects current state by checking for files
2. Executes appropriate phase skill
3. Manages loop termination based on performance goals

## Tool Skills (Used in PHASE 2 and PHASE 3)

These skills invoke Truffle/Graal profiling tools:

**Phase 2 (Broad Discovery)**: Run on entire benchmark to discover issues
**Phase 3 (Targeted Investigation)**: Run with filters focused on specific theory/function

**High-level profiling**:
- `profiling-with-cpu-sampler` - Time-based profiling, tier breakdown (T0/T1/T2)
- `tracing-execution-counts` - Execution frequency measurement
- `detecting-performance-warnings` - Finds optimization barriers (virtual calls, type checks)

**Compilation analysis**:
- `tracing-compilation-events` - JIT compilation monitoring
- `tracing-inlining-decisions` - Inlining behavior analysis
- `detecting-deoptimizations` - Deoptimization tracking (goal: zero in steady-state)

**Deep analysis**:
- `profiling-memory-allocations` - Allocation tracking
- `analyzing-compiler-graphs` - BGV dump analysis with Seafoam/bgv2json

**Documentation**:
- `fetching-truffle-documentation` - Access Truffle API docs

## Key Concepts

### Fermi Verification
Every tool skill requires "Fermi verification" - pre-calculate expected results before running tools, then validate actual results are within expectations. This prevents silent tool failures and garbage data.

### Tool Output Storage
Tools save outputs to `tool-outputs/` directory with naming pattern: `tool-outputs/{tool-name}-{benchmark}.txt`

### State Files
- `BENCHMARK_BASELINE.md` - Benchmark descriptions, timing data, performance expectations
- `PERFORMANCE_THEORIES.md` - Testable theories with verification plans and status tracking

### Theory Fields

**Status Values:**
- `pending` - Theory generated, awaiting action
- `verified` - Theory proven with Phase 3 targeted profiling
- `falsified` - Theory disproven (in Phase 3 or after implementation failed)
- `fixed` - Fix implemented and benchmarks show improvement
- `fix-failed` - Fix attempted but reverted due to regression/no improvement

**Evidence Quality:**
- `DEFINITIVE` - Specific tool output with line numbers, node types, exact barriers
- `CIRCUMSTANTIAL` - General patterns or observations without specific tool confirmation

**Deeper Investigation Decision:**
- `SKIP` - Implement directly (definitive evidence + low-risk fix)
- `REQUIRED` - Run Phase 3 targeted profiling (circumstantial evidence OR high-risk fix)

**Fix Complexity:**
- `LOW-RISK` - <10 lines, single file, localized change, easy to revert
- `HIGH-RISK` - Architectural changes, multi-file, expensive to implement

## Common Patterns

### Truffle Performance Issues
The skills are designed to detect and fix:
- Missing primitive specializations (boxing overhead)
- Uncached method calls (virtual calls instead of direct)
- Improper `@TruffleBoundary` usage
- Frame access causing materialization
- Object shape instability (deoptimization loops)
- Failed escape analysis (allocations not eliminated)

### Profiling Approach

**Phase 2 (Broad Discovery)**:
- Run cpusampler, memtracer, IGV on entire benchmark
- Identifies: hot functions, allocation patterns, compilation issues
- Generates theories with evidence quality assessment

**Phase 3 (Targeted Investigation)** - CONDITIONAL:
- Only for theories with "Deeper Investigation: REQUIRED"
- Same tools as Phase 2 but with filters on specific function/theory
- Example: `--vm.Djdk.graal.MethodFilter='*specificFunction*'`
- Provides definitive evidence before expensive implementation

**Benchmarks as Validation**:
- Phase 4 always runs benchmarks after implementation
- Benchmarks prove whether fix actually improved performance
- If fix fails, optionally run additional profiling to diagnose

### Compilation Tiers
- T0 (Interpreter) - Should be <10% for hot functions
- T1 (First-tier compiled) - Transitional
- T2 (Fully optimized) - Should be >80% for hot functions

## Testing Plugin Changes

To test changes to skills:
1. Edit skill definitions in `skills/` directory
2. Update `.claude-plugin/plugin.json` if needed
3. Restart Claude Code: `claude --plugin-dir /path/to/cc-truffle-performance-plugin`
4. Verify with `/help` command

## Skill Invocation

Skills are model-invoked (Claude selects based on context). The plugin provides specialized domain knowledge for Truffle performance optimization.
