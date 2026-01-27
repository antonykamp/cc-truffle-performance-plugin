---
name: broad-performance-investigation
description: "[PHASE 2] Broad profiling and code analysis to identify optimization opportunities. Entry: BENCHMARK_BASELINE.md exists AND (no PERFORMANCE_THEORIES.md OR all theories fixed). Outputs: PERFORMANCE_THEORIES.md with evidence quality assessment. Next: PHASE 3 (deep-performance-investigation) or PHASE 4 based on 'Deeper Investigation' decision."
---

# Broad Performance Investigation

Systematically analyzes language implementation code to generate testable performance hypotheses. Produces a prioritized list of theories ready for verification with profiling tools.

## What This Skill Does

1. **Loads benchmark data** - Reads BENCHMARK_BASELINE.md and timing results
2. **Analyzes performance gaps** - Compares actual vs expected performance
3. **Profiles with basic tools** - Runs cpusampler (hot functions/tiers), memorytracer (allocations), IGV overview (compiler optimization)
4. **Performs systematic code analysis** - Examines implementation for anti-patterns
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

**Step 3**: Run initial profiling to guide analysis:
- `profiling-with-cpu-sampler` - Identify hot functions and tier distribution
- `profiling-memory-allocations` - Detect allocation hotspots
- `analyzing-compiler-graphs` - Get IGV overview of optimization patterns

**Step 4**: Execute systematic code analysis following [WORKFLOW.md](WORKFLOW.md), using profiling data to prioritize

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
- **DEFINITIVE**: Specific tool output with line numbers, node types, or exact barriers
  - Example: Performance warning "virtual call at CallNode.java:42"
  - Example: IGV shows CommitAllocationNode in specific function
  - Example: Deopt trace shows specific line causing transfers
- **CIRCUMSTANTIAL**: General patterns or observations without specific tool confirmation
  - Example: "Function X is 30% of time" (but WHY is unknown)
  - Example: "Allocations detected" (but WHICH ones matter?)
  - Example: Code pattern suggests issue (but no tool evidence)

**Deeper Investigation Logic:**
- **SKIP** if: DEFINITIVE evidence + LOW-RISK fix
- **REQUIRED** if: CIRCUMSTANTIAL evidence OR HIGH-RISK fix

## Analysis Categories

### Implementation Issues
- Missing specializations in operations/nodes
- Uncached method calls or library usage
- Improper boundary annotations
- Frame access patterns causing materialization

**Tool Skills for Verification**:
- `detecting-performance-warnings` - Find optimization barriers
- `profiling-with-cpu-sampler` - Identify hot functions
- `tracing-execution-counts` - Verify frequencies

### Configuration Issues
- Bytecode DSL settings
- Compilation thresholds
- Boxing elimination configuration
- Missing optimization flags

**Tool Skills for Verification**:
- `tracing-compilation-events` - Check compilation behavior
- `analyzing-compiler-graphs` - Examine IR optimizations

### Architectural Issues
- Data structure choices causing allocation
- Object shape instability
- Recursive patterns without proper caching
- Control flow preventing optimization

**Tool Skills for Verification**:
- `profiling-memory-allocations` - Track allocation patterns
- `detecting-deoptimizations` - Find instabilities
- `analyzing-compiler-graphs` - Deep IR analysis

## Systematic Code Analysis

**Use profiling data (cpusampler, memorytracer, IGV) to prioritize which code areas to analyze first.**

### Step 1: Language Definition & Bytecode Configuration
```
Check @GenerateBytecode annotation settings:
- [ ] Boxing elimination enabled for primitives?
- [ ] Uncached interpreter enabled?
- [ ] Appropriate tier thresholds?
```

### Step 2: ALL Operations/Nodes
```
For EACH @Operation or Node class:
- [ ] Has primitive specializations (int, long, double)?
- [ ] Uses @Cached for method lookups?
- [ ] Proper @TruffleBoundary on slow paths?
- [ ] No virtual calls in hot paths?
```

### Step 3: Runtime Data Structures
```
For each runtime type (Object, Array, Class, Function):
- [ ] Shape stability maintained?
- [ ] Inline caching for property access?
- [ ] Efficient storage (primitives not boxed)?
```

### Step 4: Frame and Variable Access
```
- [ ] Slot access uses FrameSlotKind properly?
- [ ] No unnecessary materialization?
- [ ] Stable frame sizes?
```

### Step 5: Library and Interop Usage
```
- [ ] All @CachedLibrary has limit parameter?
- [ ] @ExportLibrary on appropriate classes?
- [ ] Interop calls cached?
```

## Priority Levels

- **Priority 1 (Critical)**: Blocks optimization entirely (missing @Cached, virtual calls in hot paths)
- **Priority 2 (High)**: Significant degradation (missing specializations, boxing overhead)
- **Priority 3 (Medium)**: Noticeable impact (suboptimal caching limits, allocation patterns)
- **Priority 4 (Low)**: Minor optimizations (code style, minor inefficiencies)

## Output Format

```markdown
# Performance Theories

Generated: [Date]
Focus: [User-selected focus]
Benchmark: [Benchmark name]

## Theory 1: [Title]

**Status**: pending
**Source**: [code location or performance gap]
**Category**: Implementation
**Severity**: Critical

**Evidence Quality**: DEFINITIVE
**Evidence**:
- Performance warning: "virtual call at CallNode.java:42"
- cpusampler: CallNode consumes 30% execution time
- Tier distribution: 60% T0, 30% T1, 10% T2

**Fix Complexity**: LOW-RISK (add @Cached, 2 lines, single file)
**Deeper Investigation**: SKIP (definitive evidence + low-risk fix)

**Rationale**: Virtual call prevents inlining and specialization

---

## Theory 2: [Title]

**Status**: pending
**Source**: [code location or performance gap]
**Category**: Architectural
**Severity**: High

**Evidence Quality**: CIRCUMSTANTIAL
**Evidence**:
- cpusampler: ArithmeticNode consumes 25% execution time
- Code review: No primitive specializations found
- memtracer: No allocation data for this function

**Fix Complexity**: HIGH-RISK (requires specialization architecture, 50+ lines, multiple files)
**Deeper Investigation**: REQUIRED (circumstantial evidence + high-risk fix)

**Verification Tools**:
1. `detecting-performance-warnings` → Check for specific optimization barriers
2. `analyzing-compiler-graphs` → Examine what compiler actually does

**Expected Evidence**: Boxing nodes in IR, or type check warnings

**Rationale**: Missing specializations likely causing boxing overhead

---
```

## Integration with Other Skills

**Prerequisite Skills**:
- `establishing-benchmark-baseline` → Provides BENCHMARK_BASELINE.md

**Successor Skills**:
- `deep-performance-investigation` → Verifies theories with profiling tools
- `implementing-performance-fixes` → Implements and validates fixes for verified theories

**Tool Skills Referenced**:
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

See [WORKFLOW.md](WORKFLOW.md) for detailed analysis procedures.
