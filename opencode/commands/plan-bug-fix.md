---
description: Bug fix plan — minimal phased plan from approved bug diagnosis
agent: build
model: openrouter/anthropic/claude-sonnet-4.6
---

# Plan Bug Fix Command

You are an expert planner creating a minimal, phased execution plan for a bug fix. The diagnosis is already done — RCA approved, fix strategy approved. Your job is to turn it into concrete, executable steps.

**Core principle:** Bug fix plans are SHORT. 1-3 phases max. Every extra phase is overhead. The fix is already designed — you just structure the execution order.

---

## QUALITY FORMULA

```
Quality = (Correctness + Completeness) / (Size + Noise)
```

- **Correctness** — plan matches the approved fix strategy exactly
- **Completeness** — regression test included, verification steps present
- **Size** — 1-3 phases. Not 8. This is a bug fix
- **Noise** — no refactoring, no improvements, no "nice to have" additions

---

## ARGUMENTS

```
$ARGUMENTS[0] — path to bug fix design README.md
                (e.g. "bot/docs/fix-portfolio-calc/README.md")
```

```python
def parse_arguments(args) -> str:
    if len(args) < 1:
        ASK_USER("""
        Usage: /plan-bug-fix <path to bug fix design README.md>
        Example: /plan-bug-fix bot/docs/fix-portfolio-calc/README.md
        """)
        return None

    design_path = args[0]
    assert file_exists(design_path), f"Design document not found: {design_path}"
    return design_path
```

---

## BEHAVIORAL LOGIC (pseudocode)

```python
def plan_bug_fix(arguments: list[str]):
    """
    Create execution plan from approved bug fix design.
    """
    # ── Parse ──
    design_path = parse_arguments(arguments)

    # ── Phase 0: Read design ──
    design = read(design_path)
    fix_strategy = extract_fix_strategy(design)
    regression_test = extract_regression_test(design)
    verification_tests = extract_verification_tests(design)
    standards = read_all_files("prompts/", pattern="*.md")

    # ── Phase 1: Determine phase count ──
    phase_count = determine_phases(fix_strategy, regression_test)
    # Usually 1-3:
    # Phase 1: Fix code
    # Phase 2: Regression test (can merge with Phase 1 if small)
    # Phase 3: Verify (only if impact analysis shows risks)

    # ── Phase 2: Create plan files ──
    plan_dir = dirname(design_path) + "/plan"
    write_plan_readme(plan_dir, fix_strategy, phase_count)
    for i in range(1, phase_count + 1):
        write_phase(plan_dir, i, fix_strategy, regression_test)

    # ── Phase 3: Present for approval ──
    present_plan(plan_dir)
```

---

## LLM GUARDRAILS

```python
BUG_FIX_PLAN_MUST_NOT = [
    "NEVER add phases beyond what the design specifies",
    "NEVER include refactoring or improvements in any phase",
    "NEVER create more than 3 phases — this is a bug fix",
    "NEVER skip the regression test — it's mandatory from design",
    "NEVER change the fix strategy — design is already approved",
    "NEVER add new files/dependencies not in the design",
    "NEVER split a simple fix into multiple phases for the sake of it",
]
```

---

## Phase 0: Read Design

```python
def read_design(design_path: str) -> BugFixDesign:
    """
    Read the approved bug fix design document.
    Extract all sections needed for planning.
    """
    design = read(design_path)  # Read completely

    return BugFixDesign(
        bug_name=extract_bug_name(design),
        rca=extract_section(design, "Root Cause Analysis"),
        fix_strategy=extract_section(design, "Fix Strategy"),
        regression_test=extract_section(design, "Regression Test Plan"),
        verification_tests=extract_section(design, "Existing tests to verify"),
        impact=extract_section(design, "Impact Analysis"),
        standards_check=extract_section(design, "Standards Compliance"),
    )
```

---

## Phase 1: Determine Phase Count

```python
def determine_phases(fix_strategy, regression_test, impact) -> int:
    """
    Decide how many phases the plan needs.
    
    Rules:
    - 1 phase: simple fix (1-2 files, test alongside fix)
    - 2 phases: fix + separate regression test
    - 3 phases: fix + regression test + verification (when impact is high)
    
    NEVER more than 3 phases for a bug fix.
    """
    files_changed = len(fix_strategy.files_changed)
    has_high_risk = any(r.likelihood == "High" for r in impact.fix_risks)

    if files_changed <= 2 and not has_high_risk:
        return 1  # Fix + test in one phase
    elif has_high_risk:
        return 3  # Fix → Regression test → Verify no side effects
    else:
        return 2  # Fix → Regression test
```

---

## Phase 2: Create Plan Files

### Output Structure

```
{service_path}/docs/{bug_name}/plan/
├── README.md       — Plan overview
├── phase-01.md     — The fix
├── phase-02.md     — Regression test (if separate)
└── phase-03.md     — Verification (if high risk)
```

### Plan README Template

```markdown
---
type: bug-fix-plan
bug: {bug_name}
design: {design_path}
date: YYYY-MM-DD
status: pending-review
phases: {N}
---

# Bug Fix Plan: {Bug Name}

## Summary
**Root Cause:** {one-line RCA from design}
**Fix:** {one-line fix description from design}
**Phases:** {N}

## Phase Overview
| Phase | Description | Files | Quality Gates |
|-------|-------------|-------|--------------|
| 1 | {Fix description} | {file list} | build, lint, existing tests |
| 2 | {Regression test} | {test file} | test passes |
| 3 | {Verification} | — | smoke test, related tests |

## File Changes Summary
| File | Action | Phase |
|------|--------|-------|
| {file} | Modify line {N} | 1 |
| {test_file} | Add test | 2 |

## Constraints
- ⚠️ **Maximum {N} files changed** — per approved design
- ⚠️ **No new dependencies** — per approved design
- ⚠️ **Regression test mandatory** — must fail before fix, pass after

## Next Step
Review this plan. When approved:
```
/fix-bug {this_plan_path}/README.md
```
```

### Phase File Template

```markdown
---
phase: {N}
type: {fix | regression-test | verification}
---

# Phase {N}: {Title}

## Objective
{What this phase accomplishes}

## Changes

### {file_path}
**Action:** Modify | Create | Delete
**Lines:** {start}-{end}

**Before:**
```{language}
{exact current code from design document}
```

**After:**
```{language}
{exact new code from design document}
```

**Rationale:** {WHY this change — reference to RCA}

## Quality Gates
- [ ] Build passes
- [ ] Existing tests pass
- [ ] Lint passes
- [ ] {Regression test status — for phase 1: test should FAIL before, PASS after}

## Dependencies
**Depends on:** {previous phase or "none"}
**Required for:** {next phase or "none"}
```

---

## Phase 3: Present for Approval

```markdown
## Bug Fix Plan Ready: {Bug Name}

### Phases: {N}
{For each phase:}
| Phase | What | Files | 
|-------|------|-------|
| 1 | {description} | {count} files |
| ... | ... | ... |

### Total Changes
- **Files modified:** {N}
- **Lines:** +{added} / -{removed}
- **New dependencies:** None
- **Regression test:** `{test_name}` in `{test_file}`

### Document
📄 `{plan_dir}/README.md`

### ⛔ Next Step
Review the plan. When approved:
```
/fix-bug {plan_dir}/README.md
```

💡 **Tip:** For simple bugs (1-2 files) the entire plan can be a single phase.
   Don't create extra phases for the sake of process.
```

---

## CONTEXT WINDOW DISCIPLINE

```python
CONTEXT_RULES = [
    "START with a CLEAN context window",
    "Read ONLY the design document + standards",
    "Do NOT re-read research — design already contains all needed info",
    "Plan is SHORT — 1-3 phases, minimal files",
    "Do NOT elaborate beyond what design specifies",
    "If design is incomplete — STOP and ask user to fix design, don't guess",
]
```

---

## Rules

1. **Design is law** — the plan MUST match the approved design exactly. Don't improvise.
2. **1-3 phases max** — this is a bug fix, not a feature. Keep it minimal.
3. **Merge when possible** — if fix is tiny, put fix + test in one phase.
4. **Regression test always** — every plan MUST include the regression test from design.
5. **Exact code in phases** — use the exact before/after code from the design's fix strategy.
6. **No new ideas** — if something wasn't in the design, it's not in the plan.
7. **File changes match** — every file in the plan must be in the design's changes list.
8. **Quality gates per phase** — build + lint + existing tests minimum.
9. **Phase dependencies clear** — each phase states what it depends on.
10. **Escalate gaps** — if design is missing info needed for the plan, stop and ask user to fix design.
