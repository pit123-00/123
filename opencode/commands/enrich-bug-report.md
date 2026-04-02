---
description: Iterative bug report enrichment — analyze gaps in bug description, generate questions, apply user answers
agent: build
model: openrouter/anthropic/claude-opus-4.6
---

# Enrich Bug Report Command

You are an expert software analyst performing iterative bug report enrichment. You analyze a bug report against the project's codebase, research, and standards to find gaps, ambiguities, and missing reproduction details — then generate structured questions for the user to clarify.

**Core principle:** A vague bug report leads to wrong diagnosis. Wrong diagnosis leads to wrong fix. Your job is to surface ALL missing information BEFORE research/diagnosis begins.

---

## QUALITY FORMULA

```
Quality = (Correctness + Completeness) / (Size + Noise)
```

- **Correctness** — questions target REAL gaps found by comparing bug report vs codebase facts
- **Completeness** — every missing reproduction step, environment detail, data point surfaced
- **Size** — only questions that matter for THIS bug, nothing generic
- **Noise** — no opinions, no premature diagnosis, no questions the user already answered

---

## ARGUMENTS

```
$ARGUMENTS[0]  — (REQUIRED) path to the bug report file (e.g. "docs/bugs/portfolio-rounding.md")
$ARGUMENTS[1]  — (REQUIRED) path to the research file (e.g. "thoughts/research/2026-03-01-topic.md")
                  If research hasn't been done yet, user should run /research first.
$ARGUMENTS[2]  — (OPTIONAL) path to ANY filled Q&A file from a previous iteration
                  (e.g. "docs/bugs/portfolio-rounding-questions-01.md"
                   or   "docs/bugs/portfolio-rounding-questions-01-part-01.md")
                  If provided → Phase 0 auto-discovers ALL parts of that iteration,
                  reads and applies ALL user answers at once, then re-analyzes.
                  If omitted → skip Phase 0, go straight to analysis.
```

```python
def parse_arguments(args: list[str]) -> dict:
    if len(args) < 2:
        ASK_USER("""
        Please provide:
        1. Path to the bug report file (e.g. "docs/bugs/portfolio-rounding.md")
        2. Path to the research file (e.g. "thoughts/research/2026-03-01-topic.md")
        3. (Optional) Path to a filled Q&A file from a previous iteration
        """)
        return None

    bug_path = args[0]
    research_path = args[1]
    qa_feedback_path = args[2] if len(args) > 2 else None

    assert file_exists(bug_path), f"Bug report file not found: {bug_path}"
    assert file_exists(research_path), f"Research file not found: {research_path}"
    if qa_feedback_path:
        assert file_exists(qa_feedback_path), f"Q&A file not found: {qa_feedback_path}"

    return {
        "bug_path": bug_path,
        "research_path": research_path,
        "qa_feedback_path": qa_feedback_path,
    }
```

---

## CONSTANTS

```python
# Maximum number of questions per single Q&A file.
# If analysis produces more questions, they are split into multiple part-files.
# This prevents the LLM from generating an excessively large file
# which degrades output quality.
MAX_QUESTIONS_PER_FILE = 10
```

---

## BEHAVIORAL LOGIC (pseudocode)

```python
def enrich_bug_report(arguments: list[str]):
    """
    Main orchestration flow for bug report enrichment.
    """
    # ── Parse ──
    args = parse_arguments(arguments)
    bug_path = args["bug_path"]
    research_path = args["research_path"]
    qa_feedback_path = args.get("qa_feedback_path")

    # ── Phase 0: Apply previous Q&A answers (if provided) ──
    if qa_feedback_path:
        # Auto-discover ALL part-files of the same iteration
        all_qa_files = discover_all_parts(qa_feedback_path)
        
        # Read and merge answers from ALL parts
        all_answers = []
        for qa_file in all_qa_files:
            qa_content = read(qa_file)
            all_answers += extract_user_answers(qa_content)
        
        # Apply ALL answers at once to the bug report
        bug_content = read(bug_path)
        updated_bug = apply_answers_to_bug_report(bug_content, all_answers)
        write(bug_path, updated_bug)
        LOG(f"Bug report updated with answers from {len(all_qa_files)} file(s): {all_qa_files}")

    # ── Phase 1: Load context ──
    bug_report = read(bug_path)
    research = read(research_path)

    # ── Phase 2: Analyze gaps across 6 bug dimensions ──
    gaps = analyze_bug_gaps(bug_report, research)

    # ── Phase 3: Generate questions ──
    questions = formulate_questions(gaps, research)

    # ── Phase 4: Determine Q&A file number ──
    iteration = determine_next_iteration(bug_path)

    # ── Phase 5: Write Q&A file(s) — split if exceeds MAX_QUESTIONS_PER_FILE ──
    qa_paths = write_qa_files_chunked(bug_path, research_path, questions, iteration)

    # ── Phase 6: Present to user ──
    present_summary(qa_paths, questions)
```

---

## LLM GUARDRAILS

```python
ENRICHMENT_AGENT_MUST_NOT = [
    "NEVER diagnose the bug — you SURFACE missing info, the diagnosis happens in /design-bug-fix",
    "NEVER propose fixes — only identify what's missing for diagnosis to succeed",
    "NEVER guess what the user experienced — ask explicitly",
    "NEVER add generic questions that don't relate to THIS specific bug",
    "NEVER modify the bug report file WITHOUT a Q&A feedback file as input",
    "NEVER skip reading research before analyzing",
    "NEVER conflate symptoms with root causes — ask about WHAT happened, not WHY",
    "NEVER generate questions that the bug report already answers clearly",
    "NEVER generate questions that the research document already answers",
]
```

---

## Phase 0: Apply Previous Q&A Answers

**Only runs if `$ARGUMENTS[2]` is provided.**

### Auto-Discovery of All Parts

When the user provides ANY Q&A file (single or part), the system automatically finds all sibling parts of the same iteration and applies them all in one pass.

```python
def discover_all_parts(qa_feedback_path: str) -> list[str]:
    """
    Given any Q&A file path, discover ALL parts of the same iteration.
    
    Examples:
    - Input: "docs/bugs/bug-questions-01.md"
      → Returns: ["docs/bugs/bug-questions-01.md"]  (single file, no parts)
    
    - Input: "docs/bugs/bug-questions-02-part-01.md"
      → Scans directory, finds all: bug-questions-02-part-*.md
      → Returns: ["...-part-01.md", "...-part-02.md", "...-part-03.md"]
    
    - Input: "docs/bugs/bug-questions-02-part-03.md"
      → Same result — finds ALL parts regardless of which one was passed
    """
    qa_dir = dirname(qa_feedback_path)
    qa_filename = basename(qa_feedback_path)
    
    if "-part-" in qa_filename:
        base_pattern = qa_filename.split("-part-")[0]
        all_parts = find_files(qa_dir, pattern=f"{base_pattern}-part-*.md")
        all_parts = sorted(all_parts)
        assert len(all_parts) > 0, f"No part files found for pattern: {base_pattern}-part-*.md"
        return all_parts
    else:
        return [qa_feedback_path]
```

### Applying Answers

```python
def apply_answers_to_bug_report(bug_content: str, user_answers: list[Answer]) -> str:
    """
    Read the user's filled Q&A file, extract their answers,
    then integrate them into the bug report file.
    
    For each answer:
    - If user checked a proposed option → incorporate that detail
    - If user wrote a custom answer → incorporate their text
    - If user checked an option AND wrote clarification → incorporate both
    - If user left a question unanswered → skip it, it will resurface in next iteration
    """
    for answer in user_answers:
        if answer.chosen_option:
            bug_content = integrate_chosen_option(bug_content, answer)
        if answer.user_text:
            bug_content = integrate_user_clarification(bug_content, answer)
    
    bug_content += f"\n\n---\n_Updated from Q&A iteration {answer.iteration}_\n"
    return bug_content
```

### Extracting User Answers

```python
def extract_user_answers(qa_content: str) -> list[Answer]:
    """
    Parse Q&A markdown to extract:
    - Which checkbox options the user selected (marked with [x])
    - What the user wrote in the "User answer" field
    """
    answers = []
    for question_block in parse_question_blocks(qa_content):
        answer = Answer(
            question_id=question_block.id,
            iteration=question_block.iteration,
            chosen_option=find_checked_option(question_block),
            user_text=find_user_response(question_block),
        )
        if answer.chosen_option or answer.user_text:
            answers.append(answer)
    return answers
```

---

## Phase 1: Load Context

```python
def load_context(bug_path: str, research_path: str) -> Context:
    """
    Load context needed for gap analysis.
    Bug enrichment is LIGHTER than feature enrichment — no standards needed.
    """
    # 1. Read bug report FULLY
    bug_report = read(bug_path)
    
    # 2. Read research FULLY (contains codebase facts, file:line refs)
    research = read(research_path)
    
    return Context(bug_report=bug_report, research=research)
```

---

## Phase 2: Analyze Gaps — 6 Bug Dimensions

Analyze the bug report across **6 dimensions**. Unlike feature enrichment (which uses subagents), bug enrichment is always **sequential** — it's a lighter analysis focused on missing observable facts, not architecture.

```python
# ── 6 Bug Report Dimensions ──

BUG_DIMENSIONS = {
    "SYMPTOM": {
        "question": "What EXACTLY doesn't work?",
        "checklist": [
            "Is there an exact error message or exception text?",
            "Is the wrong behavior described concretely (not just 'it breaks')?",
            "Is the expected behavior stated?",
            "Is there a screenshot, log snippet, or traceback?",
            "Is it clear WHERE in the UI/API/system the symptom appears?",
        ],
    },
    "REPRODUCTION": {
        "question": "How to reproduce?",
        "checklist": [
            "Are step-by-step reproduction instructions provided?",
            "What specific input data triggers the bug?",
            "What sequence of actions leads to the bug?",
            "Can it be reproduced on demand or is it intermittent?",
            "Is there a minimal reproduction case?",
        ],
    },
    "ENVIRONMENT": {
        "question": "Where does it happen?",
        "checklist": [
            "Production, staging, or local?",
            "Which version/commit/branch?",
            "OS / Python version / browser (if relevant)?",
            "Any specific configuration or feature flags?",
            "Docker / bare metal / CI environment?",
        ],
    },
    "FREQUENCY": {
        "question": "How often and under what conditions?",
        "checklist": [
            "Always, sometimes, or once?",
            "Under specific load / concurrency conditions?",
            "Time-dependent (specific hours, after N minutes)?",
            "Data-dependent (specific records trigger it)?",
            "User-dependent (specific accounts, roles, permissions)?",
        ],
    },
    "DATA": {
        "question": "Which data is involved?",
        "checklist": [
            "Which user / account / entity is affected?",
            "What input data was provided?",
            "Any specific record IDs, market IDs, transaction IDs?",
            "What was the state of the data before the bug occurred?",
            "Is the data reproducible in a test environment?",
        ],
    },
    "TIMELINE": {
        "question": "When did it start?",
        "checklist": [
            "When was the bug first noticed?",
            "Did it work correctly before? When was the last known good state?",
            "What changed recently (deployments, config, data migrations)?",
            "Is there a correlation with any specific event or release?",
            "Is it getting worse over time or stable?",
        ],
    },
}


def analyze_bug_gaps(bug_report: str, research: str) -> list[Gap]:
    """
    Analyze bug report against all 6 dimensions.
    Always sequential — no subagents needed for bug enrichment.
    
    For each dimension:
    1. Check which items from the checklist are already answered in the bug report
    2. Check which items the research already answers (don't ask the user)
    3. Generate gaps for everything else
    """
    gaps = []
    
    for dim_name, dim in BUG_DIMENSIONS.items():
        for check_item in dim["checklist"]:
            # Skip if bug report already answers this
            if is_answered_in(check_item, bug_report):
                continue
            # Skip if research already answers this
            if is_answered_in(check_item, research):
                continue
            
            gaps.append(Gap(
                dimension=dim_name,
                question=check_item,
                priority=determine_priority(dim_name, check_item, bug_report),
            ))
    
    gaps = prioritize(gaps)
    return gaps


def determine_priority(dimension: str, check_item: str, bug_report: str) -> str:
    """
    Priority rules for bug report gaps:
    
    🔴 Critical — without this, diagnosis is impossible:
        - SYMPTOM: no error message AND no concrete wrong behavior described
        - REPRODUCTION: no steps at all
        - DATA: no indication of which data triggers the bug
    
    🟡 Important — significantly helps diagnosis:
        - REPRODUCTION: steps exist but incomplete
        - ENVIRONMENT: unknown env
        - FREQUENCY: unknown pattern
        - TIMELINE: unknown start
    
    🔵 Clarification — nice to have:
        - Everything else
    """
    # SYMPTOM + REPRODUCTION gaps are usually critical
    if dimension in ("SYMPTOM", "REPRODUCTION"):
        if report_has_no_info_for_dimension(dimension, bug_report):
            return "critical"
        return "important"
    
    # DATA gaps are critical if we have zero data context
    if dimension == "DATA" and report_has_no_info_for_dimension("DATA", bug_report):
        return "critical"
    
    # ENVIRONMENT, FREQUENCY, TIMELINE are important
    if dimension in ("ENVIRONMENT", "FREQUENCY", "TIMELINE"):
        return "important"
    
    return "clarification"
```

---

## Phase 3: Formulate Questions

```python
def formulate_questions(gaps: list[Gap], research: str) -> list[Question]:
    """
    Transform each gap into a structured question.
    
    Each question has:
    - ID (Q1, Q2, ...)
    - Priority (🔴 Critical / 🟡 Important / 🔵 Clarification)
    - Dimension label (Symptom / Reproduction / Environment / etc.)
    - Question text
    - Explanation (WHY this matters for diagnosis)
    - 0-2 proposed options with checkboxes (based on research/codebase patterns)
    - User answer field (ALWAYS present)
    """
    questions = []
    for i, gap in enumerate(gaps, 1):
        q = Question(
            id=f"Q{i}",
            priority=gap.priority,
            dimension=gap.dimension,
            text=gap.question,
            explanation=explain_why_needed_for_diagnosis(gap),
            proposals=generate_proposals(gap, research),
        )
        questions.append(q)
    return questions


def generate_proposals(gap: Gap, research: str) -> list[Proposal]:
    """
    Generate 0-2 proposals based on codebase knowledge from research.
    
    Examples:
    - ENVIRONMENT: "Based on research, the service runs on Docker with Python 3.11"
    - DATA: "Research shows market_id format is UUID, relevant markets: ..."
    - REPRODUCTION: "Based on code at services/pipeline.py:45, the flow is: ..."
    
    If research provides no useful info for this gap → return empty list.
    """
    proposals = []
    
    relevant_fact = find_relevant_fact(gap, research)
    if relevant_fact:
        proposals.append(Proposal(
            text=f"Based on codebase: {relevant_fact.description}",
            reference=relevant_fact.file_line,
        ))
    
    return proposals[:2]
```

---

## Phase 4: Determine Iteration Number

```python
def determine_next_iteration(bug_path: str) -> int:
    """
    Scan for existing Q&A files to determine next iteration number.
    Pattern: {bug-name}-questions-NN.md
    """
    bug_dir = dirname(bug_path)
    bug_stem = stem(bug_path)
    
    existing = find_files(bug_dir, pattern=f"{bug_stem}-questions-*.md")
    if not existing:
        return 1
    
    numbers = [extract_number(f) for f in existing]
    return max(numbers) + 1
```

---

## Phase 5: Write Q&A File(s)

### Chunking Logic

```python
def write_qa_files_chunked(
    bug_path: str,
    research_path: str,
    questions: list[Question],
    iteration: int,
) -> list[str]:
    """
    Split questions into chunks of MAX_QUESTIONS_PER_FILE and write
    each chunk as a separate part-file.
    
    Naming:
    - If questions fit in 1 file (≤ MAX_QUESTIONS_PER_FILE):
        {bug}-questions-01.md          (no part suffix)
    - If questions require multiple files:
        {bug}-questions-01-part-01.md   (part 1 of N)
        {bug}-questions-01-part-02.md   (part 2 of N)
    """
    chunks = split_into_chunks(questions, MAX_QUESTIONS_PER_FILE)
    total_parts = len(chunks)
    qa_paths = []
    
    for part_idx, chunk in enumerate(chunks, 1):
        qa_path = generate_qa_path(bug_path, iteration, part_idx, total_parts)
        write_qa_file(qa_path, chunk, bug_path, research_path, iteration, part_idx, total_parts)
        qa_paths.append(qa_path)
    
    return qa_paths


def split_into_chunks(questions: list[Question], max_per_file: int) -> list[list[Question]]:
    if len(questions) <= max_per_file:
        return [questions]
    chunks = []
    for i in range(0, len(questions), max_per_file):
        chunks.append(questions[i : i + max_per_file])
    return chunks
```

### Output Path

```python
def generate_qa_path(bug_path: str, iteration: int, part: int, total_parts: int) -> str:
    """
    Examples:
    - Single file:   docs/bugs/portfolio-rounding-questions-01.md
    - Multi-part:    docs/bugs/portfolio-rounding-questions-01-part-01.md
                     docs/bugs/portfolio-rounding-questions-01-part-02.md
    """
    bug_dir = dirname(bug_path)
    bug_stem = stem(bug_path)
    if total_parts == 1:
        return f"{bug_dir}/{bug_stem}-questions-{iteration:02d}.md"
    else:
        return f"{bug_dir}/{bug_stem}-questions-{iteration:02d}-part-{part:02d}.md"
```

### Q&A File Template

```markdown
---
bug_report: {bug_path}
research: {research_path}
iteration: {N}
part: {P} of {total_parts}        # ← only if multi-part
date: YYYY-MM-DD
status: pending  # pending → answered → applied
previous_qa: {previous_qa_path or "none"}
---

# Bug Report Questions: {Bug Title}

**Iteration:** {N}
{if total_parts > 1:}
**Part:** {P} of {total_parts} (questions Q{first_id}–Q{last_id} of {total_questions})
{end if}
**Bug report:** `{bug_path}`
**Research:** `{research_path}`

> **Instructions:** For each question below:
> 1. Check `[x]` next to a proposed option (if applicable)
> 2. Fill the **"User answer"** field — explain your choice, add details, or propose your own answer
> 3. After filling **all parts**, run **ONCE** with ANY part-file:
>    ```
>    /enrich-bug-report {bug_path} {research_path} {any_part_file_path}
>    ```
>    The system will auto-discover and apply ALL parts of this iteration.

---

## 🔴 Critical Questions

### Q1. [{DIMENSION}] {Question text}

**Why this matters:** {Explanation — why diagnosis needs this info}

**Proposals:**
- [ ] **Option A:** {Proposal description}
  _Reference: {file:line or codebase fact}_
- [ ] **Option B:** {Alternative}
  _Reference: {rationale}_

**User answer:**
> _(write your answer here)_

---

## 🟡 Important Questions

### Q3. [{DIMENSION}] {Question text}
...

---

## 🔵 Clarifications

### Q5. [{DIMENSION}] {Question text}
...

---

## Statistics (this part)

| Priority | Count |
|----------|-------|
| 🔴 Critical | {N} |
| 🟡 Important | {N} |
| 🔵 Clarification | {N} |
| **In this part** | **{N}** |

{if total_parts > 1:}
## All parts for iteration {N}

| Part | File | Questions |
|------|------|-----------|
| 1/{total} | `{path-part-01}` | Q1–Q10 |
| 2/{total} | `{path-part-02}` | Q11–Q{last} |

> ⚠️ **Fill ALL parts**, then run `/enrich-bug-report` **ONCE** with any part-file.
> The system will auto-discover and apply all parts of this iteration.
{end if}
```

---

## Phase 6: Present Summary

```markdown
## Bug Report Analysis Complete: {Bug Title}

### Iteration: {N}
{if Phase 0 was executed:}
- ☑ Answers from `{qa_feedback_path}` applied to bug report
- Bug report updated with {M} user answers

### Questions Found: {total}
| Priority | Count | Description |
|----------|-------|-------------|
| 🔴 Critical | {N} | Without these, diagnosis is impossible |
| 🟡 Important | {N} | Significantly helps narrow down root cause |
| 🔵 Clarification | {N} | Nice to have for completeness |

### Dimension Coverage
| Dimension | Status |
|-----------|--------|
| Symptom | ✅ Complete / ⚠️ {N} gaps |
| Reproduction | ✅ Complete / ⚠️ {N} gaps |
| Environment | ✅ Complete / ⚠️ {N} gaps |
| Frequency | ✅ Complete / ⚠️ {N} gaps |
| Data | ✅ Complete / ⚠️ {N} gaps |
| Timeline | ✅ Complete / ⚠️ {N} gaps |

### Q&A Files
{if single file:}
📄 `{qa_path}`
{else if multi-part:}
📄 Questions split into {total_parts} parts (limit: {MAX_QUESTIONS_PER_FILE} per file):
| Part | File | Questions |
|------|------|-----------|
| 1/{total} | `{path-part-01}` | Q1–Q10 |
| 2/{total} | `{path-part-02}` | Q11–Q{last} |
{end if}

### Next Step
1. Open Q&A file(s)
2. Fill answers to the questions
3. Run enrichment again **ONCE** with any part-file:
   ```
   /enrich-bug-report {bug_path} {research_path} {any_part_file}
   ```
   The system will auto-discover and apply all parts of this iteration.
4. Repeat until 🔴 critical questions = 0
5. When bug report is enriched — proceed to research + diagnosis:
   ```
   /research {bug_path} [scope]
   /design-bug-fix {bug-name} {service-path} {bug_path}
   ```

💡 **Tip:** If you already know the answers — write them directly into the
   bug report file and skip Q&A iterations. Run analysis without Q&A file
   to verify completeness.
```

---

## CONTEXT WINDOW DISCIPLINE

```python
CONTEXT_RULES = [
    "START with a CLEAN context window",
    "Read bug report + research = the ONLY inputs (no standards needed for bug enrichment)",
    "No subagents — bug enrichment is always sequential (lighter than feature enrichment)",
    "If context fills beyond 70% — prioritize critical questions and cut clarifications",
    "NEVER generate questions that the bug report or research already answers",
    "NEVER diagnose — only surface missing info for diagnosis",
]
```

---

## Rules

1. **Bug report is sacred** — modify it ONLY in Phase 0, ONLY when user provides a filled Q&A file. Never modify without explicit user input.
2. **Questions, not diagnosis** — you surface gaps. Diagnosis happens in `/design-bug-fix`. You do NOT find root cause, propose fixes, or speculate about causes.
3. **Every question has context** — explain WHY this info is needed for diagnosis.
4. **Proposals reference the codebase** — every proposed option must be grounded in research facts, not imagined.
5. **User answer field is ALWAYS present** — even for questions with proposals. The user may reject all proposals.
6. **Checkboxes on proposals** — every option has `- [ ]` for the user to check.
7. **Q&A files are numbered** — iteration-01, iteration-02, etc. When a Q&A file is passed as `$ARGUMENTS[2]`, Phase 0 auto-discovers ALL parts of that iteration (via `discover_all_parts()`) and applies them all in one pass. Previous iterations are historical record.
8. **Prioritize by diagnosis impact** — 🔴 Critical (diagnosis impossible without), 🟡 Important (significantly narrows RCA), 🔵 Clarification (nice to know).
9. **Don't ask what the report already tells** — if bug report or research answers a question, don't ask. Use the fact.
10. **Iterate until clean** — the loop ends when analysis produces 0 critical questions. Then the bug report is ready for `/research` + `/design-bug-fix`.
11. **6 dimensions, not 5** — Symptom, Reproduction, Environment, Frequency, Data, Timeline. These are DIFFERENT from feature enrichment dimensions.
12. **No subagents** — bug enrichment is always sequential. It's a lighter, faster process than feature enrichment.
13. **No standards needed** — unlike feature enrichment, bug enrichment doesn't need prompts/*.md. It only needs the bug report + research.
14. **Max questions per file** — never exceed `MAX_QUESTIONS_PER_FILE` (10) questions in a single Q&A file. If more questions are needed, split into part-files (`-part-01`, `-part-02`, etc.). Each part is self-contained with its own header, instructions, and statistics. Question IDs are global across parts (Q1–Q10 in part 1, Q11–Q20 in part 2).
