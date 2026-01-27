# Documentation Fixes for Truffle Performance Plugin

This document contains drafted fixes for the 10 critical issues identified in the review.

## File 1: broad-performance-investigation/SKILL.md

### Fix #1: Explicit Phase 2 Tool Set

**Location**: After line 8, replace section "What This Skill Does"

```markdown
## What This Skill Does

This skill has two distinct activities:

### Activity 1: Static Code Analysis (No Tool Execution)
Systematically examines implementation code to identify potential performance anti-patterns:
- Missing `@Cached` annotations causing virtual dispatch
- Missing primitive specializations causing boxing
- Improper `@TruffleBoundary` usage blocking optimization
- Frame access patterns causing materialization
- Object shape instabilities

### Activity 2: Broad Profiling (Phase 2 Tool Set)
Runs THREE profiling tools on the ENTIRE benchmark to gather initial evidence:

1. **`profiling-with-cpu-sampler`** - WHERE time is spent
   - Identifies hot functions consuming most execution time
   - Shows tier distribution (T0/T1/T2) revealing compilation issues
   - Command: `<launcher> --cpusampler --cpusampler.ShowTiers=true <benchmark>`

2. **`profiling-memory-allocations`** - WHAT allocations occur
   - Detects allocation hotspots and patterns
   - Identifies excessive object creation
   - Command: `<launcher> --memtracer <benchmark>`

3. **`analyzing-compiler-graphs`** - HOW compiler optimizes
   - Generates IGV dump showing optimization decisions
   - Reveals escape analysis failures, boxing, virtual calls
   - Command: `<launcher> --vm.Dgraal.Dump=:2 <benchmark>`

**Phase 2 Tool Characteristics:**
- Run on ENTIRE benchmark (no filters)
- Broad discovery mode (find all potential issues)
- Outputs saved to `tool-outputs/cpu-sampler-[benchmark].txt`, etc.

### Activity 3: Theory Generation
Combines static analysis findings with tool evidence to generate theories with:
- Evidence quality assessment (DEFINITIVE vs CIRCUMSTANTIAL)
- Cost-benefit analysis (verification strategy)
- Recommended Phase 3 tools if deeper investigation needed
```

### Fix #7: Static Code Analysis Patterns

**Location**: New section after "What This Skill Does"

```markdown
## Static Code Analysis Checklist

Before running profiling tools, examine the codebase for these patterns:

### 1. Missing @Cached Annotations (Implementation Issue)

**Search pattern:**
```java
@Specialization
public Object doSomething(...) {
    someLibrary.method();  // ← Uncached call
    helperFunction();      // ← Uncached call
}
```

**What to look for:**
- `@Specialization` methods that call other methods
- Method calls without corresponding `@Cached` parameters
- Library usage without caching

**Impact**: Virtual dispatch instead of direct calls, prevents inlining

**Example theory source**: "AddNode.java:42 calls Math.sqrt() without @Cached"

---

### 2. Missing Primitive Specializations (Architectural Issue)

**Search pattern:**
```java
@Specialization
public Object doAdd(Object a, Object b) {
    // Only generic Object version, no int/long/double versions
}
```

**What to look for:**
- Nodes with ONLY Object-typed specializations
- No `doInt`, `doLong`, `doDouble` variants
- Arithmetic/comparison operations without primitive paths

**Impact**: Boxing overhead for numeric operations

**Example theory source**: "ArithmeticNode only has Object specializations"

---

### 3. Improper @TruffleBoundary Usage (Implementation Issue)

**Search pattern:**
```java
@TruffleBoundary
public Object execute(VirtualFrame frame) {
    // Boundary on hot path execute method
}
```

**What to look for:**
- `@TruffleBoundary` on `execute()` methods
- Boundary annotations on frequently-called operations
- Unnecessary boundaries on Truffle-safe code

**Impact**: Prevents inlining and partial evaluation

**Example theory source**: "CallNode.execute() has @TruffleBoundary"

---

### 4. Frame Access Causing Materialization (Implementation Issue)

**Search pattern:**
```java
public Object execute(VirtualFrame frame) {
    MaterializedFrame materialized = frame.materialize();
    // Or: storing frame in field, escaping frame
}
```

**What to look for:**
- Explicit `frame.materialize()` calls
- Frame stored in object fields
- Frame passed to non-inlinable methods
- Frame escaping local scope

**Impact**: Forces expensive frame materialization

**Example theory source**: "ClosureNode stores VirtualFrame in field"

---

### 5. Recursive Patterns Without Caching (Architectural Issue)

**Search pattern:**
```java
@Specialization
public Object doRecursive(...) {
    return this.execute(...);  // Self-recursion without proper caching
}
```

**What to look for:**
- Nodes calling their own execute methods
- Recursive dispatch without `@Cached` or `DirectCallNode`
- Tree-walking patterns without caching

**Impact**: Repeated compilation, failed optimization

**Example theory source**: "EvalNode has uncached recursive execute calls"

---

### 6. Suboptimal Configuration (Configuration Issue)

**Files to check:**
- Language registration (`@TruffleLanguage` annotations)
- Compilation thresholds
- Bytecode DSL settings
- Optimization flags in build configuration

**What to look for:**
```java
@TruffleLanguage.Registration(
    bytecodeInterpreterSwitch = false  // ← Should be true for performance
)
```

**Impact**: Missed optimization opportunities

**Example theory source**: "Language registration disables bytecode interpreter switch"

---

## How to Use This Checklist

**For each category:**
1. Use Grep/Read tools to search for patterns in implementation files
2. Document locations of potential issues (file:line)
3. Note the pattern type and expected impact
4. Add to theory list with "Source: Static analysis"

**After static analysis:**
- Proceed to Activity 2 (broad profiling) to gather tool evidence
- Correlate tool outputs with static analysis findings
- Generate theories combining both sources
```

### Fix #4: Evidence Quality Rubric

**Location**: Replace lines 73-82 with expanded rubric

```markdown
## Evidence Quality Assessment

Evidence quality determines confidence level, NOT whether to verify. Use this rubric:

### DEFINITIVE Evidence

Evidence is DEFINITIVE when tool output points to a **specific, actionable location**:

✅ **Performance warning with exact location**
- Example: `[engine] perf warning: virtual call at CallNode.java:42`
- Tool: `detecting-performance-warnings`
- Why definitive: Exact file, line, and issue type

✅ **Deoptimization with quantified frequency**
- Example: `Deoptimization: 847 transfers to interpreter from LoopNode.java:23`
- Tool: `detecting-deoptimizations`
- Why definitive: Exact location and frequency measured

✅ **Compiler graph shows specific problematic node**
- Example: `CommitAllocationNode found in optimized IR for calculateSum() at bytecode offset 15`
- Tool: `analyzing-compiler-graphs`
- Why definitive: Exact function and optimization failure point

✅ **Compilation failure with reason**
- Example: `Function 'parse' not compiled: call to blacklisted method String.format`
- Tool: `tracing-compilation-events`
- Why definitive: Exact function and blocking reason

✅ **Inlining failure with specific callsite**
- Example: `Not inlined: CallTarget.call() at Parser.java:156 (polymorphic call site)`
- Tool: `tracing-inlining-decisions`
- Why definitive: Exact location and failure reason

---

### CIRCUMSTANTIAL Evidence

Evidence is CIRCUMSTANTIAL when it identifies a **general pattern without specific tool confirmation**:

⚠️ **Hot function without root cause**
- Example: `cpusampler: ArithmeticNode.execute() consumes 35% execution time`
- Why circumstantial: Tells WHERE time is spent, not WHY it's slow

⚠️ **Code pattern suggests issue**
- Example: `Static analysis: No primitive specializations found in ArithmeticNode`
- Why circumstantial: Pattern exists, but no proof it matters for performance

⚠️ **Tier distribution anomaly without cause**
- Example: `cpusampler: calculateSum() shows 60% T0, 30% T1, 10% T2`
- Why circumstantial: Shows compilation issue exists, not what's blocking it

⚠️ **Allocation suspected but unconfirmed**
- Example: `Code creates ArrayList in loop, likely causing allocations`
- Why circumstantial: Logical deduction, but no memtracer confirmation

⚠️ **Expected optimization not confirmed**
- Example: `Code should benefit from escape analysis, but performance suggests it's not happening`
- Why circumstantial: Hypothesis, not measurement

---

### Rubric Summary

**Rule of thumb:**
- **DEFINITIVE** = Can point to file:line with tool output showing the problem
- **CIRCUMSTANTIAL** = Suspect an issue exists, but need targeted profiling to confirm details

**Important**: Evidence quality is about confidence, NOT about whether to skip verification. The verification strategy decision is based on cost-benefit (see next section).
```

### Fix #2: Cost-Benefit Framing for Verification Strategy

**Location**: Replace lines 84-88 with cost-benefit analysis

```markdown
## Verification Strategy Decision (Phase 3 or Direct Implementation?)

After assessing evidence quality, decide whether to run Phase 3 (deep-performance-investigation) or skip to Phase 4 (implementing-performance-fixes).

**This is a COST-BENEFIT decision, not an evidence confidence decision.**

### Cost-Benefit Framework

Calculate:
1. **Implementation Cost** (time to code + test the fix)
2. **Investigation Cost** (time to run Phase 3 targeted profiling)

**Decision rule:**
- If `Implementation Cost < Investigation Cost` → **VERIFY-FIRST**
- If `Implementation Cost > Investigation Cost` → **IMPLEMENT-FIRST**

---

### IMPLEMENT-FIRST Strategy (Deeper Investigation: SKIP)

**When to use:**
- Implementation is trivial: <10 lines, single file, <5 min to code
- Fix is easily reversible (add/remove annotation, one-line change)
- Running benchmarks is faster than running 3+ profiling tools

**Rationale**: Cheaper to implement the fix and let benchmarks prove/disprove than to run extensive profiling.

**Example scenarios:**

✅ **Scenario 1: Add @Cached annotation**
- Fix: Add `@Cached("create()") LibraryNode lib` to specialization (2 lines)
- Implementation cost: 2 minutes
- Investigation cost: Run 3 tools with filters (15 minutes)
- Decision: **IMPLEMENT-FIRST** (7.5x faster)

✅ **Scenario 2: Add @TruffleBoundary**
- Fix: Add annotation to method (1 line)
- Implementation cost: 1 minute
- Investigation cost: Run IGV dump + analyze (20 minutes)
- Decision: **IMPLEMENT-FIRST** (20x faster)

✅ **Scenario 3: Fix obvious configuration**
- Fix: Change `bytecodeInterpreterSwitch = true` (1 line)
- Implementation cost: 30 seconds
- Investigation cost: Run compilation traces (10 minutes)
- Decision: **IMPLEMENT-FIRST** (20x faster)

---

### VERIFY-FIRST Strategy (Deeper Investigation: REQUIRED)

**When to use:**
- Implementation is complex: 50+ lines, multiple files, architectural changes
- Fix is expensive to implement (30+ min to code)
- Fix is risky or hard to revert (refactoring, API changes)
- Evidence is circumstantial (need to confirm problem exists)

**Rationale**: Invest in targeted profiling to avoid wasting time implementing the wrong fix.

**Example scenarios:**

✅ **Scenario 1: Add primitive specialization architecture**
- Fix: Add `doInt`, `doLong`, `doDouble` specializations with conversion logic (50+ lines)
- Implementation cost: 2 hours
- Investigation cost: Run compiler graphs + performance warnings (20 minutes)
- Decision: **VERIFY-FIRST** (6x faster to verify than implement wrong fix)

✅ **Scenario 2: Refactor frame access pattern**
- Fix: Restructure closure implementation to avoid frame materialization (100+ lines)
- Implementation cost: 4 hours
- Investigation cost: Run memtracer + IGV with filters (30 minutes)
- Decision: **VERIFY-FIRST** (8x faster to verify than implement wrong fix)

✅ **Scenario 3: Architectural change for shape stability**
- Fix: Redesign object representation to prevent deoptimization loops (multiple files)
- Implementation cost: 8 hours
- Investigation cost: Run deopt tracing + compiler graphs (30 minutes)
- Decision: **VERIFY-FIRST** (16x faster to verify than implement wrong fix)

✅ **Scenario 4: Circumstantial evidence needs confirmation**
- Evidence: "cpusampler shows 35% time in ArithmeticNode, but WHY is unclear"
- Investigation cost: Run performance warnings + compiler graphs to find root cause (20 minutes)
- Decision: **VERIFY-FIRST** (need to understand problem before proposing fix)

---

### Decision Template for Theories

When generating theories, include this assessment:

```markdown
**Implementation Cost**: [LOW (<10 lines, <5 min) | MEDIUM (10-50 lines, 5-30 min) | HIGH (50+ lines, 30+ min)]
**Investigation Cost**: [LOW (1 tool, <10 min) | MEDIUM (2-3 tools, 10-30 min) | HIGH (4+ tools, 30+ min)]
**Verification Strategy**: [IMPLEMENT-FIRST | VERIFY-FIRST]
**Deeper Investigation**: [SKIP | REQUIRED]

**Rationale**: [Brief cost-benefit explanation]
```

**Example 1 (IMPLEMENT-FIRST):**
```markdown
**Implementation Cost**: LOW (add @Cached annotation, 2 lines, single file)
**Investigation Cost**: MEDIUM (run performance warnings + inlining traces, 15 min)
**Verification Strategy**: IMPLEMENT-FIRST
**Deeper Investigation**: SKIP

**Rationale**: Implementing fix takes 2 minutes, running tools takes 15 minutes. Faster to implement and benchmark.
```

**Example 2 (VERIFY-FIRST):**
```markdown
**Implementation Cost**: HIGH (add primitive specializations, 50+ lines, 3 files, 2 hours)
**Investigation Cost**: MEDIUM (run compiler graphs + performance warnings, 20 min)
**Verification Strategy**: VERIFY-FIRST
**Deeper Investigation**: REQUIRED

**Rationale**: Fix requires 2 hours of implementation. Spending 20 minutes to verify problem exists avoids wasting time if theory is wrong.
```
```

### Fix #8: Phase 2 Tool Output Analysis

**Location**: Replace "Quick Start" section (lines 32-50)

```markdown
## Quick Start

**Step 0**: Check prerequisites
- [ ] `BENCHMARK_BASELINE.md` exists
- [ ] Benchmark timing data available
- [ ] At least one benchmark shows performance gap vs expectations

**Step 1**: Ask user for analysis focus
Use AskUserQuestion to determine scope:
- All issues (comprehensive)
- Critical & high-impact only
- Implementation issues only
- Configuration issues only
- Architectural issues only

**Step 2**: Load benchmark baseline and identify performance gaps
- Read `BENCHMARK_BASELINE.md`
- Compare actual vs expected performance
- Identify which benchmarks show largest gaps
- Note: Focus profiling on slowest benchmarks first

**Step 3**: Execute static code analysis
Follow the "Static Code Analysis Checklist" (see section below):
- Grep for missing `@Cached` annotations
- Grep for missing primitive specializations
- Grep for improper `@TruffleBoundary` usage
- Check frame access patterns
- Review language configuration files
- Document findings with file:line references

**Step 4**: Run Phase 2 profiling tools (broad discovery mode)

For the SLOWEST benchmark (largest performance gap):

A. Run cpu-sampler:
```bash
<launcher> --cpusampler --cpusampler.ShowTiers=true --cpusampler.OutputFile=tool-outputs/cpu-sampler-[benchmark].txt <benchmark>
```
Expected output: Histogram showing hot functions and tier distribution

B. Run memory tracer:
```bash
<launcher> --memtracer --memtracer.OutputFile=tool-outputs/memtracer-[benchmark].txt <benchmark>
```
Expected output: Allocation histogram

C. Run IGV dump:
```bash
<launcher> --vm.Dgraal.Dump=:2 --vm.Dgraal.DumpPath=tool-outputs/igv-[benchmark] <benchmark>
```
Expected output: BGV dump files in tool-outputs/

**Important**: Save all outputs to `tool-outputs/` directory for later reference.

**Step 5**: Analyze tool outputs and correlate with static analysis

**5a. Analyze cpu-sampler output:**
- [ ] Identify top 3-5 hot functions (highest Total Time %)
- [ ] Check tier distribution for each hot function
  - T2 >80% = Good (compiled and optimized)
  - T0 >30% = Problem (stuck in interpreter)
  - T1 >30% = Problem (compiled but not optimizing)
- [ ] Cross-reference hot functions with static analysis findings
  - Did static analysis identify issues in these functions?
- [ ] Document: "Function X consumes Y% time with Z% T0/T1/T2"

**5b. Analyze memtracer output:**
- [ ] Identify allocation hotspots (highest allocation count/size)
- [ ] Determine allocation types (which classes/objects)
- [ ] Check if allocations are in hot functions from cpu-sampler
- [ ] Cross-reference with static analysis
  - Did code analysis predict these allocations?
- [ ] Document: "Function X allocates Y objects of type Z per iteration"

**5c. Analyze IGV dump:**
- [ ] Use bgv2json or Seafoam to analyze graphs
- [ ] Check for optimization failures:
  - CommitAllocationNode (escape analysis failed)
  - BoxNode / UnboxNode (boxing not eliminated)
  - Virtual calls (not devirtualized)
  - Type checks (instanceof, casts)
- [ ] Cross-reference with hot functions
  - Do optimization failures occur in hot paths?
- [ ] Document: "Function X shows CommitAllocationNode at bytecode offset Y"

**Step 6**: Generate theories combining static analysis + tool evidence

For each potential issue identified:

A. **Determine if there's evidence from BOTH static analysis AND tools**:
- DEFINITIVE: Tool output with specific file:line reference
- CIRCUMSTANTIAL: Only static analysis OR only general tool pattern

B. **Assess implementation cost**:
- Count lines of code to change
- Count files affected
- Estimate complexity (trivial annotation vs architectural refactor)

C. **Decide verification strategy**:
- IMPLEMENT-FIRST: If implementation cost < investigation cost
- VERIFY-FIRST: If implementation cost > investigation cost

D. **Specify Phase 3 tools if VERIFY-FIRST**:
- List which profiling tools to run with targeted filters
- Specify what evidence would confirm the theory

**Step 7**: Prioritize theories by severity
- Critical: Blocks optimization entirely (>50% time, stuck in T0)
- High: Significant degradation (20-50% time, optimization failures)
- Medium: Noticeable impact (10-20% time, allocation patterns)
- Low: Minor optimizations (<10% time)

**Step 8**: Write PERFORMANCE_THEORIES.md
- Include all theories with complete fields
- Sort by priority (Critical → High → Medium → Low)
- Ensure each theory has verification strategy and rationale
```

---

## File 2: implementing-performance-fixes/SKILL.md

### Fix #3: Run All Benchmarks Once

**Location**: Replace Step 3 and Step 4 (lines 32-37)

```markdown
**Step 3**: Run ALL benchmarks to validate fix effectiveness

- Execute every benchmark from `BENCHMARK_BASELINE.md` with same parameters
- Collect timing data for each benchmark
- Compare against baseline performance data
- Goal: Comprehensive validation across entire benchmark suite

**Example command pattern:**
```bash
for benchmark in queens mandelbrot nbody sieve ...; do
    <launcher> <harness> $benchmark <outer-iterations> <inner-iterations>
done
```

**Step 4**: Evaluate results using validation criteria

Apply the "Fix Validation Criteria" (see section below):
- [ ] All benchmarks pass correctness checks
- [ ] Calculate net performance delta across all benchmarks
- [ ] Check for regressions (any benchmark >10% slower)
- [ ] Verify benchmark reliability (variance <5%)

**Decision:**
- ✅ **ACCEPT** if criteria met → Update BENCHMARK_BASELINE.md, mark theory "fixed"
- ❌ **REJECT** if criteria failed → Revert fix, mark theory status appropriately
- 🔄 **RE-RUN** if variance high → Run benchmarks again to rule out flakiness
```

### Fix #9: Phase 4 Validation Criteria

**Location**: New section after "Quick Start"

```markdown
## Fix Validation Criteria

After running all benchmarks, apply these criteria to decide whether to accept or reject the fix.

### Criterion 1: Correctness ✓

**All benchmarks must pass correctness checks.**

- [ ] Every benchmark produces expected output
- [ ] No crashes, exceptions, or errors
- [ ] Verification logic passes for each benchmark

**If any benchmark fails correctness:**
- 🛑 **IMMEDIATELY REVERT** the fix
- Mark theory status: `fix-failed`
- Note: "Fix broke correctness in benchmark X"
- Do NOT proceed to performance evaluation

---

### Criterion 2: Net Performance Improvement ✓

**Overall performance must improve across the benchmark suite.**

**Calculate net delta:**
```
For each benchmark i:
  delta_i = (baseline_time_i - new_time_i) / baseline_time_i * 100

Net performance delta = sum(delta_i) / num_benchmarks
```

**Acceptance threshold:**
- ✅ Net delta > 0% → Performance improved overall
- ❌ Net delta ≤ 0% → No improvement or regression

**Example:**
```
| Benchmark  | Baseline | After Fix | Delta  |
|------------|----------|-----------|--------|
| queens     | 100ms    | 75ms      | +25%   |
| mandelbrot | 200ms    | 190ms     | +5%    |
| nbody      | 150ms    | 155ms     | -3.3%  |
| sieve      | 80ms     | 70ms      | +12.5% |

Net delta = (25 + 5 - 3.3 + 12.5) / 4 = +9.8% ✅ ACCEPT
```

---

### Criterion 3: No Severe Regressions ✓

**No individual benchmark should regress significantly.**

**Regression threshold: 10%**

- [ ] Check each benchmark's delta
- [ ] Flag any benchmark with delta < -10% (>10% slower)

**Decision rules:**
- ✅ No benchmarks regressed >10% → ACCEPT
- ⚠️ 1 benchmark regressed >10%, but net delta is strongly positive (>20%) → ASK USER
- ❌ 1+ benchmarks regressed >10%, net delta <20% → REJECT

**Example (ASK USER scenario):**
```
| Benchmark  | Baseline | After Fix | Delta   |
|------------|----------|-----------|---------|
| queens     | 100ms    | 40ms      | +60%    |
| mandelbrot | 200ms    | 90ms      | +55%    |
| nbody      | 150ms    | 180ms     | -20%    | ← Regression
| sieve      | 80ms     | 35ms      | +56%    |

Net delta = +37.75% (strong improvement)
But: nbody regressed by 20%

→ ASK USER: "Fix significantly improves 3 benchmarks (+57% avg) but regresses nbody by 20%. Accept fix?"
```

---

### Criterion 4: Measurement Reliability ✓

**Benchmark variance must be low enough to trust results.**

**Variance calculation:**
```
For each benchmark, if you ran multiple iterations:
  variance = (max_time - min_time) / avg_time * 100
```

**Reliability threshold: 5%**

- [ ] Calculate variance for each benchmark
- [ ] Flag any benchmark with variance >5%

**Decision rules:**
- ✅ All variances <5% → Results reliable, proceed with evaluation
- ⚠️ Some variances >5% → RE-RUN those benchmarks with more iterations
- ❌ Variances still >5% after re-run → Environment issue, ask user to investigate

**Note:** High variance indicates:
- Background processes interfering
- Thermal throttling
- Non-deterministic behavior in benchmark
- Insufficient warmup iterations

---

### Criterion 5: Expected Impact Alignment ✓

**Performance improvement should align with theory predictions.**

From the theory's "Expected Impact" field, check:
- [ ] Did benchmarks that exercise fixed code path improve?
- [ ] Did unrelated benchmarks remain stable?

**Example:**
- Theory: "Missing @Cached in ArithmeticNode causes virtual dispatch"
- Prediction: Benchmarks with heavy arithmetic (mandelbrot, nbody) should improve
- Result: mandelbrot +40%, nbody +35%, queens +2% (not arithmetic-heavy)
- ✅ Alignment confirmed

**If impact doesn't align:**
- ⚠️ Unexpected benchmarks improved → Fix may have broader impact, document findings
- ❌ Predicted benchmarks didn't improve → Fix may not address root cause, investigate further

---

## Validation Decision Matrix

| Correctness | Net Delta | Regressions | Variance | Decision |
|-------------|-----------|-------------|----------|----------|
| ❌ Fail     | -         | -           | -        | **REJECT** (revert immediately) |
| ✅ Pass     | ≤0%       | -           | <5%      | **REJECT** (no improvement) |
| ✅ Pass     | >0%       | >10%        | <5%      | **REJECT** (regression too severe) |
| ✅ Pass     | >20%      | >10% (1 BM) | <5%      | **ASK USER** (trade-off decision) |
| ✅ Pass     | >0%       | <10%        | <5%      | **ACCEPT** (clear win) |
| ✅ Pass     | Any       | Any         | >5%      | **RE-RUN** (unreliable data) |

---

## Post-Decision Actions

### If ACCEPT:
1. Keep fix in codebase
2. Update `BENCHMARK_BASELINE.md` with new timing data
3. Update theory status to `fixed` with performance impact data
4. Save tool outputs for historical reference

### If REJECT (no improvement):
1. Revert fix completely
2. Update theory status:
   - If came from Phase 3 (verified) → Mark `fix-failed` (theory was right, fix approach was wrong)
   - If skipped Phase 3 (IMPLEMENT-FIRST) → Mark `falsified` (theory was wrong)
3. **Optional**: Run additional profiling to diagnose why fix failed
4. Document lessons learned in theory notes

### If REJECT (correctness failure):
1. Revert fix immediately
2. Mark theory status: `fix-failed`
3. Note: "Fix broke correctness, needs different approach"
4. Investigate what went wrong before retrying

### If RE-RUN (high variance):
1. Check system load, close background processes
2. Increase warmup iterations
3. Increase measurement iterations
4. Run benchmarks again
5. If variance still high, ask user to investigate environment
```

### Fix #5: Clarify Status Values

**Location**: Replace "Status Values" section (lines 88-96)

```markdown
## Theory Status State Machine

Theories progress through states as they're investigated and implemented:

```
                    ┌─────────────┐
                    │   pending   │ ← Phase 2 generates theories
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
       [VERIFY-FIRST]            [IMPLEMENT-FIRST]
              │                         │
              ▼                         │
    ┌──────────────────┐                │
    │ Phase 3 running  │                │
    └────────┬─────────┘                │
             │                         │
    ┌────────┴────────┐                │
    │                 │                │
    ▼                 ▼                │
┌─────────┐     ┌──────────┐          │
│verified │     │falsified │          │
└────┬────┘     └──────────┘          │
     │                                │
     │          ┌─────────────────────┘
     │          │
     ▼          ▼
┌────────────────────┐
│   Phase 4 running  │
└─────────┬──────────┘
          │
     ┌────┴─────┐
     │          │
     ▼          ▼
┌─────────┐  ┌────────────┐
│  fixed  │  │ fix-failed │
└─────────┘  └────────────┘
```

### Status Definitions

| Status | Meaning | Next Action |
|--------|---------|-------------|
| `pending` | Theory generated by Phase 2, awaiting action | Check verification strategy: VERIFY-FIRST → Phase 3, IMPLEMENT-FIRST → Phase 4 |
| `verified` | Theory proven by Phase 3 targeted profiling | Phase 4: Implement fix |
| `falsified` | Theory disproven by Phase 3 OR fix failed after IMPLEMENT-FIRST | Archive theory, select next theory |
| `fixed` | Fix implemented, benchmarks show improvement, baseline updated | Archive theory, select next theory |
| `fix-failed` | Fix implemented but benchmarks showed no improvement or regression (came from VERIFY-FIRST path) | Revert fix, try different approach or archive theory |
| `inconclusive` | Phase 3 profiling couldn't confirm or deny theory | Try different tools or archive for later |
| `blocked` | Cannot implement due to dependencies or architectural constraints | Note blocking reason, revisit later |

### Status Transition Rules

**From `pending`:**
- → `verified` via Phase 3 (if VERIFY-FIRST and profiling confirms)
- → `falsified` via Phase 3 (if VERIFY-FIRST and profiling contradicts)
- → `fixed` via Phase 4 (if IMPLEMENT-FIRST and benchmarks improve)
- → `falsified` via Phase 4 (if IMPLEMENT-FIRST and benchmarks don't improve)
- → `inconclusive` via Phase 3 (if profiling results unclear)
- → `blocked` via Phase 4 (if implementation impossible)

**From `verified`:**
- → `fixed` via Phase 4 (if benchmarks improve)
- → `fix-failed` via Phase 4 (if benchmarks don't improve)

**From `inconclusive`:**
- → `verified` via Phase 3 (if re-run with different tools confirms)
- → `falsified` via Phase 3 (if re-run contradicts)
- → `archived` (if cannot resolve)

**Terminal states**: `fixed`, `falsified`, `fix-failed`, `blocked`

**Important distinction:**
- `falsified` = Theory was WRONG (evidence contradicted hypothesis)
- `fix-failed` = Theory was RIGHT (evidence confirmed hypothesis), but FIX APPROACH was wrong
```

---

## File 3: optimization-workflow-orchestrator/SKILL.md

### Fix #6: Theory Selection Priority

**Location**: New section after "State Detection" (after line 46)

```markdown
## Theory Selection Priority

When multiple theories match a state condition, select based on priority scoring:

### Priority Scoring Formula

```
Score = (Severity Weight) × (Impact Factor) - (Implementation Cost Factor)

Where:
- Severity Weight: Critical=1000, High=100, Medium=10, Low=1
- Impact Factor: % of execution time affected (from cpusampler)
- Implementation Cost Factor: LOW=0, MEDIUM=10, HIGH=50
```

### Selection Algorithm

**Step 1**: Filter theories by required state
- Example: All theories with `status: pending` AND `Deeper Investigation: SKIP`

**Step 2**: Calculate score for each filtered theory
```
Theory A: Critical, 60% time impact, LOW implementation cost
  Score = 1000 × 60 - 0 = 60,000

Theory B: High, 40% time impact, MEDIUM implementation cost
  Score = 100 × 40 - 10 = 3,990

Theory C: Critical, 30% time impact, HIGH implementation cost
  Score = 1000 × 30 - 50 = 29,950
```

**Step 3**: Select highest-scoring theory
- Winner: Theory A (60,000 points)

**Step 4**: Mark selected theory as "in-progress" (optional tracking field)

### Priority Examples

**Scenario 1: Multiple verified theories**
```
Theories:
- Theory 1: Critical severity, 80% time, from Phase 3 (verified)
- Theory 2: High severity, 30% time, from Phase 3 (verified)
- Theory 3: Critical severity, 50% time, from Phase 3 (verified)

Selection: Theory 1 (highest severity + impact)
```

**Scenario 2: Mix of implementation costs**
```
Theories:
- Theory A: High severity, 40% time, LOW cost (2 lines)
- Theory B: Critical severity, 45% time, HIGH cost (100+ lines)

Scores:
- Theory A: 100 × 40 - 0 = 4,000
- Theory B: 1000 × 45 - 50 = 44,950

Selection: Theory B (despite higher cost, severity + impact dominate)
```

**Scenario 3: Similar scores**
```
Theories:
- Theory X: High severity, 35% time, LOW cost
  Score: 100 × 35 - 0 = 3,500
- Theory Y: High severity, 37% time, LOW cost
  Score: 100 × 37 - 0 = 3,700

Selection: Theory Y (marginally higher impact)
Tiebreaker: If scores within 5%, choose lowest implementation cost
```

### Special Cases

**No impact data available:**
- Use estimated impact from evidence (e.g., "blocks compilation" = 100%)
- Default to impact = 50% if unknown

**Multiple theories affect same code:**
- Prefer lower-cost fix first
- Re-profile after fix to see if other theories still apply

**User request for specific theory:**
- Override priority scoring
- Process user-requested theory first
```

### Fix #5: Termination Behavior Clarification

**Location**: Replace "Termination Criteria" section (lines 59-73)

```markdown
## Termination Behavior

The orchestrator operates in **SINGLE-ITERATION mode by default**.

### Default Mode: Single Iteration

**Behavior:**
1. Detect current state (which phase to run)
2. Execute appropriate phase skill
3. After phase completes, STOP and report status
4. User decides whether to invoke orchestrator again

**Example flow:**
```
User: "Optimize performance"
→ Orchestrator: Detect state → Run Phase 1 → STOP
User: "Continue"
→ Orchestrator: Detect state → Run Phase 2 → STOP
User: "Continue"
→ Orchestrator: Detect state → Run Phase 3 → STOP
User: "Continue"
→ Orchestrator: Detect state → Run Phase 4 → STOP
User: "Continue"
→ Orchestrator: Detect state → Run Phase 2 (new iteration) → STOP
```

**Rationale:**
- Gives user control over pacing
- Allows user to inspect results after each phase
- Prevents runaway execution

---

### Continuous Mode (User-Requested)

**Activation:** User explicitly requests continuous operation
- "Optimize performance until all theories are addressed"
- "Run full optimization loop"
- "Continue until performance meets target"

**Behavior:**
1. Detect state and execute phase
2. After phase completes, re-detect state
3. If work remains, execute next phase automatically
4. Loop until terminal condition reached
5. Report final status

**Terminal conditions (stop looping):**
- ✅ **Success**: All theories processed (status = fixed/falsified/fix-failed)
- ✅ **Success**: Performance meets baseline expectations (see below)
- ⚠️ **Diminishing returns**: Last 3 fixes showed <2% improvement
- ❌ **Failure**: All recent fixes failed (last 5 attempts marked fix-failed)
- ❌ **Blocked**: Only blocked theories remain
- 🛑 **User interrupt**: User stops execution manually

---

### Performance Target Evaluation

**When to check:** After Phase 4 marks theory as "fixed"

**How to evaluate:**
1. Load `BENCHMARK_BASELINE.md`
2. For each benchmark, check if "Expected Performance" field exists
3. Compare current performance to expected
4. Calculate gap: `(expected - current) / expected × 100`

**Acceptance criteria:**
- ✅ All benchmarks within 10% of expected performance
- ✅ OR average gap across all benchmarks <5%
- ✅ OR user indicates satisfaction

**If criteria met:**
- Report success: "Performance optimization complete. All benchmarks meet expectations."
- STOP execution

**If criteria not met:**
- Continue: Loop back to Phase 2 to generate new theories

**If no expected performance data exists:**
- Cannot auto-evaluate, use other terminal conditions
- After each fix, report: "Fix applied. Performance gap unknown (no baseline expectations). Continue?"

---

### Status Reporting

**After each phase, report:**
```markdown
## Optimization Status

**Current Phase**: [Phase number and name]
**Theories Summary**:
- Pending: [count] ([verification strategy breakdown])
- Verified: [count]
- Fixed: [count]
- Falsified/Failed: [count]

**Performance Status**:
- Target: [Expected performance if known]
- Current: [Latest benchmark results]
- Gap: [Percentage]

**Next Action**: [Auto-continue in continuous mode | Wait for user in single-iteration mode]
```

---

### Mode Selection Guidance

**Use Single-Iteration mode when:**
- User is learning the workflow
- User wants to inspect each phase's output
- Debugging performance issues
- First time running on a codebase

**Use Continuous mode when:**
- User is confident in the workflow
- Running on CI/CD pipeline
- Batch processing multiple optimization passes
- User has limited time and wants hands-off operation

**Default**: Always start in Single-Iteration mode unless user explicitly requests continuous.
```

---

## File 4: deep-performance-investigation/SKILL.md

### Add Reference to Available Tools

**Location**: After "Tool Skills" table (line 54)

```markdown
## Phase 3 Tool Set (Targeted Profiling)

Phase 3 uses the SAME tools as Phase 2, but with **targeted filters** on specific functions/theories.

### How Phase 3 Differs from Phase 2

**Phase 2 (Broad Profiling):**
- Run on ENTIRE benchmark (no filters)
- Discover what issues exist
- Generate theories with evidence

**Phase 3 (Targeted Profiling):**
- Run on SPECIFIC function from theory (with filters)
- Confirm theory with focused data
- Provide definitive evidence for high-cost fixes

### Example: cpu-sampler

**Phase 2 command:**
```bash
<launcher> --cpusampler --cpusampler.ShowTiers=true <benchmark>
```
Output: ALL functions, ranked by time%

**Phase 3 command:**
```bash
<launcher> --cpusampler --cpusampler.ShowTiers=true <benchmark>
# Then filter output manually to focus on specific function from theory
```
Output: Same data, but only analyze the function mentioned in theory

### Example: Performance warnings with Method Filter

**Phase 2 command:**
```bash
<launcher> --engine.TraceCompilation --engine.TracePerformanceWarnings=all <benchmark>
```
Output: ALL warnings across entire benchmark

**Phase 3 command:**
```bash
<launcher> --engine.TracePerformanceWarnings=all \
           --vm.Djdk.graal.MethodFilter='*TheoryFunction*' \
           <benchmark>
```
Output: Only warnings for specific function mentioned in theory

### Example: IGV dump with CompileOnly

**Phase 2 command:**
```bash
<launcher> --vm.Dgraal.Dump=:2 <benchmark>
```
Output: Compiler graphs for MANY functions

**Phase 3 command:**
```bash
<launcher> --vm.Dgraal.Dump=:2 \
           --engine.CompileOnly='*TheoryFunction*' \
           <benchmark>
```
Output: Compiler graphs for ONLY the specific function

### Filter Syntax Reference

| Tool | Filter Option | Example |
|------|---------------|---------|
| Performance warnings | `--vm.Djdk.graal.MethodFilter` | `'*ArithmeticNode*'` |
| IGV dump | `--engine.CompileOnly` | `'*calculate*,*parse*'` |
| Compilation trace | `--vm.Djdk.graal.MethodFilter` | `'MyLang::execute'` |
| Deopt trace | `--engine.CompileOnly` | `'*hotFunction*'` |

**Wildcards:**
- `*` matches any characters
- Multiple filters: comma-separated list
- Case-sensitive by default

### Tool Selection from Theory

Theories generated in Phase 2 include "Verification Tools" field:

```markdown
**Verification Tools**:
1. `detecting-performance-warnings` → Check for specific optimization barriers in function X
2. `analyzing-compiler-graphs` → Examine what compiler does with function X
```

In Phase 3, run ONLY the tools listed, with filters targeting the function mentioned in the theory.
```

---

## File 5: CLAUDE.md (Project Instructions)

### Fix #10: State File Validation

**Location**: New section after "Performance Optimization Workflow"

```markdown
## State File Management and Validation

The workflow relies on two state files that are mutated throughout the process. To prevent corruption and ensure reliability:

### State Files

1. **`BENCHMARK_BASELINE.md`**
   - Created by: Phase 1
   - Updated by: Phase 4 (after successful fix)
   - Contains: Benchmark descriptions, timing data, expected performance

2. **`PERFORMANCE_THEORIES.md`**
   - Created by: Phase 2
   - Updated by: Phase 3 (mark verified/falsified), Phase 4 (mark fixed/fix-failed)
   - Contains: Theories with status, evidence, verification plans

### Required Validation Before Each Phase

**Before reading state files, validate structure:**

```markdown
## State File Validation Checklist

### BENCHMARK_BASELINE.md
- [ ] File is valid markdown (parseable)
- [ ] Contains "Benchmarks" section with entries
- [ ] Each benchmark has: Name, Description, Command, Expected Output
- [ ] Has "Actual Performance Results" section with timing table
- [ ] Timing data has columns: Benchmark, Date, Iterations, Total Time, Avg Time
- [ ] (Optional) Has "Expected Performance" section with comparable language data

If validation fails:
- **Phase 1**: Re-create file from scratch
- **Other phases**: Ask user to check file or restore from git

### PERFORMANCE_THEORIES.md
- [ ] File is valid markdown (parseable)
- [ ] Contains "Theory" section headers (## Theory N:)
- [ ] Each theory has required fields:
  - Status: [pending|verified|falsified|fixed|fix-failed|inconclusive|blocked]
  - Source: [description]
  - Category: [Implementation|Configuration|Architectural]
  - Severity: [Critical|High|Medium|Low]
  - Evidence Quality: [DEFINITIVE|CIRCUMSTANTIAL]
  - Evidence: [list of evidence points]
  - Implementation Cost: [LOW|MEDIUM|HIGH]
  - Investigation Cost: [LOW|MEDIUM|HIGH]
  - Verification Strategy: [IMPLEMENT-FIRST|VERIFY-FIRST]
  - Deeper Investigation: [SKIP|REQUIRED]
  - Rationale: [explanation]
- [ ] If Deeper Investigation = REQUIRED, has "Verification Tools" field
- [ ] No duplicate theory titles

If validation fails:
- **Phase 2**: Re-create file (safe, no prior state)
- **Phase 3/4**: STOP and ask user to repair file manually
  - Report which theory is malformed
  - Report which required field is missing
  - Suggest fix: "Add 'Status: pending' field to Theory 3"
```

### File Update Protocol

**When updating state files:**

1. **Read entire file** into memory
2. **Parse** markdown structure
3. **Locate** section/theory to update
4. **Modify** specific section (not whole file)
5. **Validate** modified content against schema
6. **Write** back to file
7. **Verify** file is still parseable after write

**Never:**
- ❌ Overwrite entire file without reading first
- ❌ Use string concatenation to build markdown (easy to create invalid syntax)
- ❌ Skip validation after writing

### Backup Recommendation

Suggest to user at Phase 1:
```
💡 Tip: This workflow will create and modify BENCHMARK_BASELINE.md and PERFORMANCE_THEORIES.md.
   Recommend committing to git after each phase to enable easy rollback if needed.
```

### Recovery from Corruption

**If state file becomes corrupted:**

1. Check git history for last valid version:
   ```bash
   git log --oneline PERFORMANCE_THEORIES.md
   git show <commit>:PERFORMANCE_THEORIES.md
   ```

2. If no git history:
   - **BENCHMARK_BASELINE.md**: Re-run Phase 1 (benchmarks still exist, just re-measure)
   - **PERFORMANCE_THEORIES.md**: Re-run Phase 2 (tool outputs saved in tool-outputs/)

3. Ask user whether to restore from git or regenerate

### Theory Conflict Resolution

**If multiple agents/users edit theories concurrently:**

Problem: Theory status conflicts (one agent marks "verified", another marks "falsified")

Solution:
- Always check theory status before updating
- If status changed unexpectedly, ask user:
  ```
  ⚠️ Warning: Theory 3 status was 'pending' when Phase 3 started,
     but is now 'verified' in the file. Did another process modify it?

     Options:
     1. Trust current file status (abandon my update)
     2. Overwrite with my status (may lose other work)
     3. Create duplicate theory with my findings
  ```

### Best Practices

1. **Commit after each phase** (user action)
2. **Validate before read** (automatic)
3. **Atomic updates** (modify specific section, not whole file)
4. **Save tool outputs** (enable regeneration without re-profiling)
5. **Version theory files** (optional: PERFORMANCE_THEORIES_v1.md, v2.md, etc.)
```

---

## Summary of Changes

This fixes document addresses all 10 critical issues:

1. ✅ **Phase 2 vs Phase 3 tools**: Explicitly documented that Phase 2 uses 3 tools broadly, Phase 3 uses 8+ tools with filters
2. ✅ **Cost-benefit framing**: Reframed "Deeper Investigation" decision as implementation cost vs investigation cost, not evidence quality
3. ✅ **Run all benchmarks**: Fixed Phase 4 to run all benchmarks once, removed "3 then all" logic
4. ✅ **Evidence quality rubric**: Added concrete examples of DEFINITIVE vs CIRCUMSTANTIAL with rule of thumb
5. ✅ **Termination behavior**: Clarified single-iteration (default) vs continuous mode, with explicit terminal conditions
6. ✅ **Theory selection priority**: Added scoring formula and selection algorithm for multiple matching theories
7. ✅ **Static analysis patterns**: Added comprehensive checklist with grep patterns and example sources
8. ✅ **Phase 2 tool analysis**: Added explicit step to analyze tool outputs and correlate with static analysis
9. ✅ **Phase 4 validation**: Added 5 criteria with decision matrix for accepting/rejecting fixes
10. ✅ **State file validation**: Added validation checklist, update protocol, and recovery procedures

## Next Steps

1. Review these fixes for accuracy
2. Apply to actual SKILL.md files in the repository
3. Test workflow with a sample Truffle language implementation
4. Iterate based on real-world usage
