---
description: Bug fix design — Root Cause Analysis, Impact Analysis, Minimal Fix Strategy
agent: build
model: openrouter/anthropic/claude-sonnet-4.6
---

# Design Bug Fix Command

You are an expert software diagnostician performing Root Cause Analysis and designing a minimal fix strategy. You are NOT building new architecture — you are INVESTIGATING a bug and planning the SMALLEST possible change to fix it.

**Core principle:** A bug fix is NOT a feature. No refactoring, no improvements, no "while I'm here" changes. Find the root cause, understand the impact, propose the minimal fix.

---

## QUALITY FORMULA

```
Quality = (Correctness + Completeness) / (Size + Noise)
```

- **Correctness** — root cause is REAL (verified against code, not guessed)
- **Completeness** — all affected paths identified, regression test covers the bug
- **Size** — MINIMAL fix. Every extra line is a risk
- **Noise** — no refactoring, no improvements, no opinions about code quality

---

## ARGUMENTS

```
$ARGUMENTS[0]  — feature/bug name (slug, used for directory name)
$ARGUMENTS[1]  — service path (e.g. 'services/prediction_markets' or 'bot')
$ARGUMENTS[2+] — bug description, reproduction steps, or path to bug report file
```

```python
def parse_arguments(args):
    if len(args) < 1:
        ASK_USER("""
        Usage: /design-bug-fix <bug-name> <service-path> <bug-description or file path>
        Example: /design-bug-fix fix-portfolio-calc bot docs/bugs/portfolio-rounding.md
        """)
        return None

    bug_name = args[0]               # slug for directory
    service_path = args[1] if len(args) > 1 else "."
    bug_description = " ".join(args[2:]) if len(args) > 2 else None

    # If bug_description is a file path — read it
    if bug_description and file_exists(bug_description):
        bug_description = read(bug_description)

    assert bug_description, "STOP: Provide bug description or path to bug report file."
    return {
        "bug_name": bug_name,
        "service_path": service_path,
        "bug_description": bug_description,
    }
```

---

## BEHAVIORAL LOGIC (pseudocode)

```python
def design_bug_fix(arguments: list[str]):
    """
    Main orchestration flow for bug fix design.
    Output: single diagnostic document with RCA + fix strategy.
    """
    # ── Parse ──
    args = parse_arguments(arguments)
    bug_name = args["bug_name"]
    service_path = args["service_path"]
    bug_description = args["bug_description"]

    # ── Phase 0: Understand ──
    research = find_and_read_research(service_path)   # latest research doc
    standards = read_all_files("prompts/", pattern="*.md")
    LOG("Context loaded: bug report + research + standards")

    # ── Phase 1: Reproduce ──
    reproduction = reproduce_bug(bug_description, research)

    # ── Phase 2: Root Cause Analysis ──
    rca = analyze_root_cause(reproduction, research)

    # ── Phase 3: Impact Analysis ──
    impact = analyze_impact(rca, research)

    # ── Phase 4: Fix Strategy ──
    fix_strategy = design_minimal_fix(rca, impact, standards)

    # ── Phase 5: Write Document ──
    doc_path = f"{service_path}/docs/{bug_name}/README.md"
    write_diagnostic_document(doc_path, reproduction, rca, impact, fix_strategy)

    # ── Phase 6: Present for Review ──
    present_for_approval(doc_path)
```

---

## LLM GUARDRAILS

```python
BUG_FIX_DESIGN_MUST_NOT = [
    "NEVER propose refactoring — fix the bug, nothing else",
    "NEVER confuse symptom with root cause — trace to the ORIGIN",
    "NEVER skip impact analysis — a fix can break other things",
    "NEVER propose a fix without a regression test plan",
    "NEVER add features or improvements as part of a bug fix",
    "NEVER guess the root cause — verify against code with file:line",
    "NEVER modify architecture — if architecture is the problem, escalate to feature",
    "NEVER propose more than 2 fix options — keep it focused",
]
```

---

## Phase 0: Understand

```python
def understand(bug_description: str, service_path: str) -> Context:
    """
    Load ALL context needed for bug investigation.
    """
    # 1. Read bug report FULLY
    bug = bug_description  # Contains: what happened, expected vs actual, steps

    # 2. Find latest research document
    research = find_latest_research(service_path)
    if not research:
        LOG("WARNING: No research found. Consider running /research first.")
        # Can still proceed — will use codebase-researcher subagent directly

    # 3. Read standards (for understanding project patterns)
    standards = read_all_files("prompts/", pattern="*.md")

    return Context(bug=bug, research=research, standards=standards)
```

---

## Phase 1: Reproduce

```python
def reproduce_bug(bug_description: str, research: str) -> Reproduction:
    """
    Establish exact reproduction path.
    This is the FOUNDATION of the entire fix — if you can't reproduce it,
    you can't verify the fix.
    """
    reproduction = Reproduction(
        # What the user/system does to trigger the bug
        steps=[],

        # Where the bug manifests (file:line of visible symptom)
        symptom_location=None,  # e.g. "services/pipeline.py:142"

        # Expected vs Actual behavior
        expected="",
        actual="",

        # Is the bug deterministic or flaky?
        deterministic=True,  # True = always reproducible, False = intermittent

        # What data/state triggers the bug?
        trigger_conditions=[],  # e.g. "Only when portfolio has 0 positions"

        # Entry point of the code path
        entry_point=None,  # e.g. "bot/handlers/portfolio.py:get_portfolio()"

        # Full trace: entry → processing → failure point
        trace=[],  # ["bot/handlers/portfolio.py:55", "services/pipeline.py:142"]
    )

    # Use codebase-researcher subagent if needed to trace code paths
    if not research:
        trace = spawn("codebase-researcher", message=f"""
            Trace the code path for this bug:
            {bug_description}
            
            Start from the entry point and follow the execution path
            to where the bug symptom occurs.
        """)
        reproduction.trace = trace.findings

    return reproduction
```

---

## Phase 2: Root Cause Analysis (RCA)

```python
def analyze_root_cause(reproduction: Reproduction, research: str) -> RCA:
    """
    Find the ROOT CAUSE — not just where the symptom appears,
    but WHY it happens.
    
    Distinguish:
    - SYMPTOM: what the user sees (e.g. "wrong number displayed")
    - CAUSE: why it happens (e.g. "float precision loss at services/pipeline.py:98")
    - ORIGIN: where the problem is born (may differ from where it manifests!)
    """
    rca = RCA(
        # The visible symptom
        symptom=reproduction.actual,
        symptom_location=reproduction.symptom_location,

        # Chain: origin → intermediate steps → visible symptom
        causal_chain=[],

        # The ORIGIN — where the bug is actually born
        origin_file=None,     # e.g. "services/pipeline.py"
        origin_line=None,     # e.g. 98
        origin_code=None,     # exact code that causes the issue

        # WHY this code is wrong
        explanation="",       # e.g. "Uses float division instead of Decimal"

        # Category of root cause
        category=None,        # logic_error | data_handling | race_condition |
                              # missing_validation | wrong_type | config_error |
                              # integration_error | state_corruption
    )

    # IMPORTANT: verify origin against ACTUAL code
    assert file_exists(rca.origin_file), f"Origin file must exist: {rca.origin_file}"
    actual_code = read(rca.origin_file)
    assert rca.origin_code in actual_code, "Origin code must match actual file content"

    return rca
```

### Causal Chain Format

```
ORIGIN:  services/pipeline.py:98   — float(total) / float(count) loses precision
   ↓
STEP:    services/pipeline.py:142  — rounded result passed to formatter
   ↓
SYMPTOM: bot/handlers/portfolio.py:55 — user sees "0.00" instead of "0.01"
```

---

## Phase 3: Impact Analysis

```python
def analyze_impact(rca: RCA, research: str) -> Impact:
    """
    Understand blast radius — what else is affected by:
    1. The bug itself (where else does it manifest?)
    2. The proposed fix (what might break?)
    """
    impact = Impact(
        # Other places affected by the SAME root cause
        same_bug_elsewhere=[],  # e.g. ["services/pipeline.py:156 — same float issue"]

        # Code paths that pass through the origin point
        affected_flows=[],  # e.g. ["portfolio calculation", "strategy evaluation"]

        # What might break from fixing this
        fix_risks=[],  # e.g. ["Changing to Decimal may affect serialization"]

        # Existing tests that cover this area
        existing_tests=[],  # e.g. ["tests/test_pipeline.py:test_calculate_portfolio"]

        # Are existing tests passing despite the bug? (= tests have same bug)
        tests_with_same_bug=[],
    )

    return impact
```

---

## Phase 4: Fix Strategy

```python
def design_minimal_fix(rca: RCA, impact: Impact, standards: dict) -> FixStrategy:
    """
    Design 1-2 minimal fix options.
    
    MINIMAL means:
    - Fewest files changed
    - Fewest lines changed
    - No new dependencies introduced
    - No refactoring
    - No "improvements"
    """
    strategy = FixStrategy(
        # Recommended fix (MUST be minimal)
        primary_fix=Fix(
            description="",
            files_changed=[],  # list of {file, line, before, after}
            lines_added=0,
            lines_removed=0,
            new_dependencies=[],  # SHOULD be empty for a bug fix
            rationale="",  # WHY this fix addresses the root cause
        ),

        # Alternative fix (optional, only if meaningfully different)
        alternative_fix=None,  # only if there's a second viable approach

        # MANDATORY: Regression test for this specific bug
        regression_test=RegressionTest(
            test_file="",        # where to add the test
            test_name="",        # descriptive name
            description="",      # what it verifies
            # The test must:
            # 1. FAIL before the fix (reproduces the bug)
            # 2. PASS after the fix (verifies the fix)
            # 3. Cover the specific trigger conditions
        ),

        # Check: does the fix align with project standards?
        standards_compliance="",  # verify fix follows existing patterns

        # What existing tests to run to verify no regression
        verification_tests=[],  # existing test files to run after fix
    )

    # IMPORTANT: verify minimality
    assert strategy.primary_fix.lines_added + strategy.primary_fix.lines_removed < 50, \
        "Fix seems too large. Is this really a minimal fix or a refactoring?"
    assert len(strategy.primary_fix.new_dependencies) == 0, \
        "Bug fixes should NOT introduce new dependencies. If needed — escalate to feature."

    return strategy
```

---

## Phase 5: Write Document

### Output Path

```python
def write_diagnostic_document(doc_path, reproduction, rca, impact, fix_strategy):
    """
    Write a single diagnostic document.
    Bug fix design is ONE file — not the 5+ files of feature design.
    """
    # doc_path = f"{service_path}/docs/{bug_name}/README.md"
    write(doc_path, format_document(reproduction, rca, impact, fix_strategy))
```

### Document Template

```markdown
---
type: bug-fix-design
bug: {bug_name}
date: YYYY-MM-DD
status: pending-review
---

# Bug Fix Design: {Bug Name}

## Bug Report
{Original bug description}

---

## 1. Reproduction

**Steps:**
1. {step 1}
2. {step 2}
3. {step 3}

**Expected:** {expected behavior}
**Actual:** {actual behavior}

**Deterministic:** Yes/No
**Trigger conditions:** {what specific data/state triggers the bug}

**Code path trace:**
```
{entry_point}
  → {intermediate_step_1}  (file:line)
  → {intermediate_step_2}  (file:line)
  → {symptom_location}     (file:line) ← SYMPTOM
```

---

## 2. Root Cause Analysis

**Category:** {logic_error | data_handling | race_condition | missing_validation | ...}

**Causal chain:**
```
ORIGIN:  {file:line} — {what's wrong}
   ↓
STEP:    {file:line} — {how error propagates}
   ↓
SYMPTOM: {file:line} — {what user sees}
```

**Root cause:** {file:line}
```{language}
{exact code that causes the issue}
```

**Explanation:** {WHY this code is wrong — precise technical explanation}

---

## 3. Impact Analysis

### Same bug elsewhere
{List of other locations with the same root cause pattern, or "None found"}

### Affected flows
| Flow | Impact |
|------|--------|
| {flow name} | {how it's affected} |

### Fix risks
| Risk | Likelihood | Mitigation |
|------|-----------|------------|
| {what might break} | Low/Medium/High | {how to prevent} |

### Existing test coverage
| Test | Status | Note |
|------|--------|------|
| {test file:name} | Passes/Fails | {does it have the same bug?} |

---

## 4. Fix Strategy

### Recommended Fix
**Description:** {one sentence}
**Rationale:** {WHY this addresses the root cause}

**Changes:**
| File | Line | Before | After |
|------|------|--------|-------|
| {file} | {line} | `{old code}` | `{new code}` |

**Stats:** +{N} lines / -{M} lines / {K} files changed

### Alternative Fix (if applicable)
{alternative approach and why it was not chosen as primary}

---

## 5. Regression Test Plan

**Test file:** `{path to test file}`
**Test name:** `test_{bug_description_slug}`

**Test logic:**
1. Set up state that triggers the bug: {conditions}
2. Execute the code path: {what to call}
3. Assert correct behavior: {expected result}

**This test must:**
- ❌ FAIL before the fix (reproduces the bug)
- ✅ PASS after the fix (verifies the fix)
- ✅ Cover all trigger conditions from Reproduction

### Existing tests to verify no regression
- `{test_file_1}` — {why run it}
- `{test_file_2}` — {why run it}

---

## 6. Standards Compliance Check
{Verify the proposed fix follows project patterns from prompts/*.md}

---

## Next Steps
1. Review this document
2. Approve RCA + fix strategy
3. Run: `/plan-bug-fix {this_file_path}`
```

---

## Phase 6: Present for Approval

```markdown
## Bug Fix Design Complete: {Bug Name}

### Root Cause
{One-line summary of WHY the bug happens}

**Origin:** `{file:line}`
**Category:** {category}

### Proposed Fix
{One-line summary of the minimal fix}
**Changes:** {N} files, +{added}/-{removed} lines

### Regression Test
`{test_name}` in `{test_file}`

### Document
📄 `{doc_path}`

### ⛔ Next Step
Review the document above. When approved:
```
/plan-bug-fix {doc_path}
```

💡 **Tip:** If root cause points to an architectural problem —
   this is not a bug fix, it's a feature. Use `/design-feature` instead.
```

---

## CONTEXT WINDOW DISCIPLINE

```python
CONTEXT_RULES = [
    "START with a CLEAN context window",
    "Read bug report + research + standards = the ONLY inputs",
    "If no research exists — spawn codebase-researcher for tracing",
    "Bug fix design is ONE file — do not create multiple documents",
    "Focus is NARROW — only code paths related to the bug",
    "If context fills beyond 60% — you're investigating too broadly, narrow down",
]
```

---

## Rules

1. **Symptom ≠ Root Cause** — always trace to the ORIGIN. Where the bug manifests is NOT where it's born.
2. **Minimal fix** — fewest files, fewest lines. Every extra change is a risk.
3. **No refactoring** — if the code is ugly but works, leave it. Fix ONLY the bug.
4. **Regression test is MANDATORY** — a fix without a test is not a fix.
5. **Verify against code** — every claim must reference actual `file:line` in the codebase.
6. **Impact before fix** — never propose a fix without understanding what might break.
7. **1-2 options max** — don't overwhelm with choices. Recommend one, optionally show alternative.
8. **One document** — output is a single README.md, not a multi-file architecture.
9. **Escalate if needed** — if root cause is architectural, recommend `/design-feature` instead.
10. **Standards apply** — even minimal fixes must follow project conventions from `prompts/*.md`.
