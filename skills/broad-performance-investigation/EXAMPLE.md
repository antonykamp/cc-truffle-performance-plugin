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