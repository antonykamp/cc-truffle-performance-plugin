# Theory Generation Workflow

Detailed procedures for systematic performance theory generation.

## Phase 0: Determine Analysis Focus

**Objective**: Understand what the user wants to investigate

### Process

1. **Check if user specified focus in the initial prompt**
   - Look for keywords: "implementation issues", "configuration", "architectural", "minor issues", "critical only", "all issues"
   - If focus is clear, proceed with that focus

2. **If focus NOT specified, ask the user**

   Use AskUserQuestion tool with:

   **Question**: "What aspects of performance should I focus on?"
   **Header**: "Analysis Focus"

   **Options**:
   1. **All issues (comprehensive)** - Analyze implementation, configuration, and architectural issues at all severity levels
   2. **Critical & high-impact only** - Focus on issues with significant performance impact, skip minor optimizations
   3. **Implementation issues only** - Focus on method-level code issues (missing specializations, caching, boundaries)
   4. **Configuration issues only** - Focus on bytecode config, compiler settings, optimization flags
   5. **Architectural issues only** - Focus on design patterns, data structure choices, overall architecture

3. **Document focus decision**
   - Record which categories to analyze
   - Record which severity levels to include
   - Note any custom criteria from user

## Phase 1: Load Benchmark and Baseline Data

**Objective**: Gather all existing performance data and expectations

### Process

1. **Load benchmark results**
   - Read `BENCHMARK_BASELINE.md` for benchmark list
   - Request user to provide timing data or locate result files
   - Parse execution times for each benchmark
   - Identify successful vs failed benchmarks

2. **Load baseline comparison data**
   - Read `BENCHMARK_BASELINE.md`:
     - Language characteristics
     - Comparable languages
     - Expected performance ranges per benchmark
     - Reference implementation times
   - Extract performance expectations

3. **Initial comparison**
   - Calculate performance ratios (actual / expected)
   - Identify significantly slower benchmarks (>2x slower than best case)
   - Identify unexpectedly slow benchmarks (>expected range)
   - Identify unexpectedly fast benchmarks (<expected range)

### Output

- Benchmark execution times loaded
- Baseline expectations loaded
- Performance gap analysis
- List of problem benchmarks

## Phase 2: Run Initial Profiling

**Objective**: Gather profiling data to guide and prioritize code analysis

### Process

1. **Run CPU Sampler** (`profiling-with-cpu-sampler`)
   - Execute with `--cpusampler --cpusampler.ShowTiers=true`
   - Identify hot functions (>5% time)
   - Check tier distribution (T0/T1/T2) for compilation issues
   - Save output to `tool-outputs/cpu-sampler-[benchmark].txt`

2. **Run Memory Tracer** (`profiling-memory-allocations`)
   - Execute with `--memtracer`
   - Identify allocation hotspots
   - Note allocation-heavy functions
   - Save output to `tool-outputs/memtracer-[benchmark].txt`

3. **Get IGV Overview** (`analyzing-compiler-graphs`)
   - Dump compiler graphs for 1-2 hot functions (from cpusampler)
   - Use `--vm.Djdk.graal.Dump=Truffle:1 --vm.Djdk.graal.MethodFilter='*hotFunction*'`
   - Quick scan for optimization patterns: indirect calls, allocations, boxing
   - Save to `compiler_graphs/` directory

### Output

- Hot function list with time percentages
- Tier distribution data (% in T0/T1/T2)
- Allocation hotspots
- High-level compiler optimization patterns
- **Prioritized code areas to analyze**

## Phase 3: Generate Performance Theories

**Objective**: Generate testable performance theories based on gaps, patterns, and user's focus areas

### Step 0: Apply Focus Filter

Based on user's specified focus:

- **If "All issues"**: Analyze all categories below
- **If "Critical & high-impact only"**: Analyze all categories, filter out Medium and Low severity
- **If "Implementation issues only"**: Focus on operations/nodes, frame access, library usage
- **If "Configuration issues only"**: Focus on language definition & bytecode config
- **If "Architectural issues only"**: Focus on runtime data structures and architectural patterns

### Step 1: Analyze Performance Gaps

For each benchmark significantly slower than baseline:
- Identify gap magnitude (2x, 3x, 10x slower)
- Consider benchmark type (recursive, allocation-heavy, arithmetic)
- Generate specific theories

### Step 2: Apply Pattern Matching

- **Recursive benchmarks** → Inlining/caching theories
- **Allocation benchmarks** → Escape analysis theories
- **Arithmetic benchmarks** → Specialization theories
- **All slow benchmarks** → Compilation effectiveness theories

### Step 3: Systematic Code Analysis (Most Important)

**Use profiling data from Phase 2 to prioritize which code areas to analyze first (hot functions, allocation sites, non-compiling code).**

**Filter analysis by user's focus:**

a. **Language definition & bytecode configuration** [CONFIGURATION]
   - Check configuration settings for optimization opportunities
   - Identify missing or suboptimal configurations

b. **ALL operations/nodes in the implementation** [IMPLEMENTATION]
   - Check EVERY operation for missing optimizations
   - Look for patterns that prevent compilation
   - Identify missing specializations or caching

c. **Runtime data structures and types** [ARCHITECTURAL + IMPLEMENTATION]
   - Analyze allocation patterns
   - Check for optimization boundaries
   - Identify inefficient data structure choices

d. **Frame and variable access patterns** [IMPLEMENTATION]
   - Analyze slot access patterns
   - Check for dynamic vs constant access
   - Identify materialization overhead

e. **Library and interop usage** [IMPLEMENTATION]
   - Check for uncached library usage
   - Analyze limit parameters on cached libraries
   - Identify missing exports or specializations

### Step 4: Record Each Anti-Pattern Found

For each anti-pattern:
- Record specific file location (file:line)
- Extract code excerpt showing the issue
- Estimate impact (Critical/High/Medium/Low)
- Categorize: Implementation vs Configuration vs Architectural
- **Filter by user's focus** - skip if outside focus areas or severity threshold

### Step 5: Prioritize Theories by Impact Level

- **Priority 1**: Critical impact (blocks optimization, causes severe slowdowns)
- **Priority 2**: High impact (significant performance degradation)
- **Priority 3**: Medium impact (noticeable but not severe)
- **Priority 4**: Low impact (minor optimizations)

### Step 6: Select ALL Verification Tools

For each theory, list ALL tools needed for complete verification:
- Match theory type to appropriate tool skills
- **CRITICAL**: List ALL tools needed for 100% proof (not just one)
- Consider verification workflow dependencies

**Tool Skill Mapping**:
| Theory Type | Primary Tool Skill | Secondary Tool Skills |
|-------------|-------------------|----------------------|
| Compilation issues | `tracing-compilation-events` | `profiling-with-cpu-sampler` |
| Specialization gaps | `detecting-performance-warnings` | `analyzing-compiler-graphs` |
| Inlining failures | `tracing-inlining-decisions` | `profiling-with-cpu-sampler` |
| Type instability | `detecting-deoptimizations` | `tracing-compilation-events` |
| Allocation overhead | `profiling-memory-allocations` | `analyzing-compiler-graphs` |
| Frequency analysis | `tracing-execution-counts` | `profiling-with-cpu-sampler` |

### Step 7: Assess Evidence Quality and Make Deeper Investigation

For EACH theory generated in Step 6, critically assess:

**A. Evidence Quality Assessment**

Review Phase 2 profiling outputs:
- Do we have specific line numbers or node types? → DEFINITIVE
- Do we only have aggregate metrics (% time, general patterns)? → CIRCUMSTANTIAL
- Did performance warnings fire for this location? → DEFINITIVE
- Did IGV show specific optimization failures? → DEFINITIVE
- Is this based on code patterns without tool confirmation? → CIRCUMSTANTIAL

**B. Fix Complexity Assessment**

Estimate the fix:
- Lines of code: <10 lines → candidate for LOW-RISK
- Files touched: Single file → candidate for LOW-RISK
- Architectural impact: Localized change → LOW-RISK, refactoring core patterns → HIGH-RISK
- Implementation time: <15 min → LOW-RISK, >30 min → HIGH-RISK
- Revert cost: Easy to undo → LOW-RISK, affects many components → HIGH-RISK

**C. Make Deeper Investigation**

Apply decision logic:
- **DEFINITIVE evidence + LOW-RISK fix** → Deeper Investigation: SKIP
- **CIRCUMSTANTIAL evidence OR HIGH-RISK fix** → Deeper Investigation: REQUIRED
- **Uncertain (50/50)** → Use AskUserQuestion to consult user

**Critical Analysis Required**: This decision determines whether to implement directly or investigate further. Be rigorous in assessing evidence quality - don't skip verification based on wishful thinking.

### Output

Prioritized list of all theories found (comprehensive analysis within user's focus areas):

For EACH theory:
- Status: pending
- Specific code location (file:line)
- Code excerpt showing the issue
- Category: Implementation/Configuration/Architectural
- Evidence Quality: DEFINITIVE or CIRCUMSTANTIAL
- Evidence: Specific tool outputs from Phase 2
- Fix Complexity: LOW-RISK or HIGH-RISK
- Deeper Investigation: SKIP or REQUIRED
- Verification tools (only if Deeper Investigation = REQUIRED)
- Expected evidence from each tool (only if Deeper Investigation = REQUIRED)
- Theory rationale and root cause
- Impact estimate (Critical/High/Medium/Low)
