---
name: broad-performance-investigation
description: "[PHASE 2] Broad profiling and code analysis to identify optimization opportunities. Entry: BENCHMARK_BASELINE.md exists AND (no PERFORMANCE_THEORIES.md OR all theories fixed). Outputs: PERFORMANCE_THEORIES.md with evidence quality assessment. Next: PHASE 3 (deep-performance-investigation) or PHASE 4 (implementing-performance-fixes)."
---

# Broad Performance Investigation

Systematically analyzes language implementation code to generate testable performance hypotheses. Produces a prioritized list of theories ready for verification with profiling tools.

## What This Skill Does

1. **Loads benchmark data** - Reads BENCHMARK_BASELINE.md and timing results
2. **Analyzes performance gaps** - Compares actual vs expected performance
3. **Performs systematic code analysis** - Examines implementation for anti-patterns
4. **Profiles with basic tools** - Runs cpusampler (hot functions/tiers), memorytracer (allocations), IGV overview (compiler optimization)
5. **Generates prioritized theories** - Each with verification plan and expected evidence
6. **Outputs theory list** - Ready for `deep-performance-investigation` skill

## Prerequisites

- **REQUIRED**: `BENCHMARK_BASELINE.md` exists (from `establishing-benchmark-baseline` skill)
- **REQUIRED**: Benchmark timing data available
- **RECOMMENDED**: Familiarity with Truffle/Graal performance patterns

## Quick Start

**Step 0**: Check if a theories file (e.g., `PERFORMANCE_THEORIES.md`) already exists in the workspace:

- If it exists with **unverified theories**, skip this workflow and continue with `deep-performance-investigation`
- If it exists but **all theories are validated and fixed**, delete it and run this skill again to generate new theories for the next optimization iteration

**Step 1**: Ask user for analysis focus using AskUserQuestion:

- All issues (comprehensive)
- Critical & high-impact only
- Implementation issues only
- Configuration issues only
- Architectural issues only

**Step 2**: Load benchmark baseline and timing results

**Step 3**: Execute systematic code analysis to identify potential performance issues based on established anti-patterns (most important step).

- Analyze the full language implementation codebase:
  - Start with RootNodes implementation of AST
  - Continue with Language implementation file
  - Analyze data structures and utility classes used in execution

**Step 4**: Run profiling to gather data for additional theories:

- Use lightweight, broad profiling tools to identify other potential issues, that may not be evident from code analysis alone.
  - `profiling-with-cpu-sampler` - Identify hot functions and tier distribution
  - `profiling-memory-allocations` - Detect allocation hotspots
  - `analyzing-compiler-graphs` - Get IGV overview of optimization patterns
- Analyze profiling outputs to find patterns indicating performance problems.

**Step 5**: Output theory list for verification

## Theory Structure

Each generated theory includes:

```
Theory: [Description of the performance issue]
Source: [Where discovered - code location, performance gap, pattern]
Evidence Quality: [DEFINITIVE / CIRCUMSTANTIAL]
Evidence: [Specific tool outputs from Phase 2 profiling]
Fix Complexity: [LOW-RISK / HIGH-RISK]
Deeper Investigation: [SKIP / REQUIRED]
Category: [Implementation / Configuration / Architectural]
Severity: [Critical / High / Medium / Low]
Verification Tools: [Only if Deeper Investigation = REQUIRED]
  - Tool 1: [skill name] → Purpose: [what to check]
  - Tool 2: [skill name] → Purpose: [what to check]
Expected Evidence: [What tool output would confirm theory]
Rationale: [Why this is expected to be an issue]
```

**Evidence Quality Guidelines:**

- Evidence quality defined by evidence strength and profiling tool confirmations
- **DEFINITIVE**: Specific tool output with line numbers, node types, or exact barriers
  - Example: Performance warning "virtual call at CallNode.java:42"
  - Example: IGV shows CommitAllocationNode in specific function
  - Example: Deopt trace shows specific line causing transfers
- **CIRCUMSTANTIAL**: General patterns or observations without specific tool confirmation
  - Example: "Function X is 30% of time" (but WHY is unknown)
  - Example: "Allocations detected" (but WHICH ones matter?)
  - Example: Code pattern suggests issue (but no tool evidence)

**Deeper Investigation Logic:**

- Assess evidence quality and fix complexity to decide if costly deeper investigation is needed
- **SKIP** if: DEFINITIVE evidence + LOW-RISK fix -> cheaper to implement directly and validate
- **REQUIRED** if: CIRCUMSTANTIAL evidence OR HIGH-RISK fix -> need tool confirmation before risking complex changes

## Analysis Categories

### Implementation Issues

- Small number of lines causing major performance hits
  - Example: Missing specializations in operations/nodes
  - Example: Uncached method calls or library usage
  - Example: Improper boundary annotations
  - Example: Frame access patterns causing materialization

**Tool Skills for further Deeper Investigation**:
- `detecting-performance-warnings` - Find optimization barriers
- `profiling-with-cpu-sampler` - Identify hot functions
- `tracing-execution-counts` - Verify frequencies

### Configuration Issues

- Suboptimal GraalVM or Truffle settings
  - Example: Bytecode DSL settings
  - Example: Compilation thresholds
  - Example: Boxing elimination configuration
  - Example: Missing optimization flags

**Tool Skills for further Deeper Investigation**:

- `tracing-compilation-events` - Check compilation behavior
- `analyzing-compiler-graphs` - Examine IR optimizations

### Architectural Issues

- Fundamental design patterns causing inefficiencies
  - Example: Lack of primitive specializations
  - Example: Data structure choices causing allocation
  - Example: Object shape instability
  - Example: Recursive patterns without proper caching
  - Example: Control flow preventing optimization

**Tool Skills for further Deeper Investigation**:

- `profiling-memory-allocations` - Track allocation patterns
- `detecting-deoptimizations` - Find instabilities
- `analyzing-compiler-graphs` - Deep IR analysis

## Priority Levels

- **Priority 1 (Critical)**: Blocks optimization entirely (missing @Cached, virtual calls in hot paths)
- **Priority 2 (High)**: Significant degradation (missing specializations, boxing overhead)
- **Priority 3 (Medium)**: Noticeable impact (suboptimal caching limits, allocation patterns)
- **Priority 4 (Low)**: Minor optimizations (code style, minor inefficiencies)

## Output Format

See [EXAMPLE.md](EXAMPLE.md) for a sample PERFORMANCE_THEORIES.md file generated by this skill.

## Integration with Other Skills

**Prerequisite Skills**:
- `establishing-benchmark-baseline` → Provides BENCHMARK_BASELINE.md

**Successor Skills**:
- `deep-performance-investigation` → Verifies theories with profiling tools
- `implementing-performance-fixes` → Implements and validates fixes for verified theories

**Tool Skills Referenced for further Deeper Investigation**:
- `profiling-with-cpu-sampler` - Time-based profiling
- `tracing-execution-counts` - Frequency analysis
- `detecting-performance-warnings` - Optimization barriers
- `tracing-compilation-events` - Compilation analysis
- `tracing-inlining-decisions` - Inlining analysis
- `detecting-deoptimizations` - Deoptimization detection
- `profiling-memory-allocations` - Memory profiling
- `analyzing-compiler-graphs` - Deep IR analysis

## Workflow

```text
1. [establishing-benchmark-baseline] → Create baseline (PHASE 1)
2. [broad-performance-investigation] → THIS SKILL (PHASE 2)
3. [deep-performance-investigation]  → Verify with tools (PHASE 3)
4. [implementing-performance-fixes]  → Implement and validate fix (PHASE 4)
5. Loop to step 2 if performance gaps remain
```
