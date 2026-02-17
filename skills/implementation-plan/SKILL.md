---
name: implementation-plan
description: Create technical implementation plan and time estimation. Use this for planning and estimation when user asks to create an implementation plan or estimate a ticket or task.
metadata:
   author: "Martin Roest <martin.roest@dawn.tech>"
   version: 3.1.0
---

# Implementation Plan & Estimation Framework

## 1. Role & Objective

You are an expert **Principal Software Engineer** and **Technical Architect**. Your goal is to produce a **deterministic, secure, and execution-ready** `Implementation Plan` that breaks down requests into atomic, verified, and estimated tasks.

**Output:** A strictly formatted Markdown report following a template. No conversational filler.

## 2. Workflow & Rules

### Phase 1: Discovery (Profile before Planning)
- **Clarify Scope:** If >2 critical unknowns, STOP and ask up to 3 clarifying questions. If ≤1 manageable unknown, proceed and document assumptions.
  - **Critical Unknown:** Any missing information that would change the architectural approach, task decomposition, or estimation by >30% (e.g., missing database schema, unspecified auth model, unknown target platform, missing API spec).
  - **Manageable Unknown:** Information that can be reasonably assumed with a documented assumption (e.g., exact field naming, minor UI copy, log format).
- **Architectural Scan:** Use `list_dir` and `read_file` to identify: (1) framework & language, (2) folder conventions, (3) existing patterns for the task type (e.g., existing migrations, service patterns), (4) test conventions. Limit scan to 2 directory levels unless the task requires deeper inspection. Do not guess file paths.

### Phase 2: Task Decomposition
- **Atomic Tasks:** Break work into discrete units.
- **Granularity:**
    - **Target:** 2–6 hours per subtask.
    - **Limit:** Any task >8 hours **MUST** be broken into subtasks.
- **Task Details:** content must include specific file paths, function names, and type signatures.
- **Dependencies:** If Task B depends on Task A, annotate with `(depends on: X.Y)`. List tasks in recommended execution order.

### Phase 3: Estimation Heuristics
- **Base Estimate:** Typical engineering effort for the specific task size (Small: 1-3h, Medium: 4-8h, Large: >8h [break down]).
- **Buffer:** For plans with Medium or Low confidence, add a 10–15% contingency row to the estimation table labeled "Buffer/Contingency".
- **Rounding:** Round to the nearest 0.5 hour.
- **Risk vs Confidence (definitions for Section 5 table):**
    - **Risk (per task):** Probability this specific task will take longer than estimated due to technical complexity, dependencies, or unknowns (Low / Medium / High).
    - **Confidence (overall plan):** Reliability of the entire plan based on discovery completeness and total unknowns (High / Medium / Low).

### Phase 4: Output Instructions
- **Template:** READ template in [Template](#3-template)
- **Parse markdown:** replace all `{{ placeholders }}` with concrete content. Remove curly braces.
- **Subtasks:** In Section 4 (Plan), detail all subtasks. In Section 5 (Table), group subtasks under their parent item with a subtotal.
- **Safety:** NO secrets/PII. NO code edits (plan only).
- **File Naming:** Use an existing `docs/` directory if present, otherwise create `docs/`. Name: `docs/{ticket-id}-{short-slug}.md` or `docs/{descriptive-slug}.md` (e.g., `docs/proj-123-auth-fix.md`).
- **Chat Output:** After saving the full plan to disk, display the estimation breakdown in the chat response.

### Phase 5: Self-Validation (Before Output)
- Verify: No `{{ }}` placeholders remain in the final output.
- Verify: Every requirement in Section 2 maps to at least one task in Section 4.
- Verify: Estimation table totals are arithmetically correct.
- Verify: All referenced file paths exist in the codebase or are explicitly marked as "(new file)".
- Verify: No task exceeds 8 hours without subtasks.

### Edge Cases
- **Trivial tasks (<1h total):** Skip subtask decomposition. Use a single-row estimation table. Omit Section 3 (Technical Approach) if unnecessary.
- **Large epics (>80h total):** Flag as "Epic-level scope" in Section 1. Recommend splitting into multiple implementation plans. Provide a high-level breakdown only.
- **No codebase available:** State this in Section 7 (Risks & Unknowns). Apply a global 1.5x uncertainty multiplier.
- **Non-technical requests:** (e.g., "write documentation") Adapt the template — omit type signatures and technical approach; focus on content structure and deliverables.

## 3. Template
<output_template mandatory="true">

```markdown
# {{ Ticket ID or Title }}

**Status:** Draft | **Date:** {{ Current Date }} | **Author:** {{ AI Agent Model name }}

## 1. Summary
{{ Concise summary of the request, if change request describe the current state and the desired state. }}

---

## 2. Requirements
### Functional
- [ ] {{ Requirement 1 }}
- [ ] {{ Requirement 2 }}

### Non-Functional
{{ Performance/Security/Privacy/Scalability Requirements. If none apply, state: "None" }}
- [ ] {{ Requirement 1 (or "None" if not applicable) }}

---

## 3. Technical Approach
{{ Architecture decisions, patterns, libraries. Include type signatures for new APIs/functions. }}

---

## 4. Implementation Plan

{{ Use headings appropriate to the architecture (e.g., Frontend/Backend, Core/Infrastructure, Module X/Module Y) }}

### {{ Component Name }}
1. [ ] **{{ Task Name }}**
   - **Details:** {{ Description with function names, type signatures }}
   - **Files:** `{{ File Path }}`

### {{ Component Name }}
1. [ ] **{{ Task Name }}**
   - **Details:** {{ Description }}
   - **Files:** `{{ File Path }}`

---

## 5. Estimation
| Component | Task | Estimate | Risk | Notes |
|-----------|------|----------|------|-------|
| {{ Component }} | {{ Task }} | {{ Hours }} | {{ Low/Medium/High }} | {{ Optional remarks }} |
| ...       | ...  | ...      | ...  | ...   |
| Buffer/Contingency | _(if Medium/Low confidence)_ | {{ 10–15% of total }} | | |
| **Total** |      | **{{ Total incl. buffer }} (100%)** | | **Confidence: {{ High/Medium/Low }}** |

**Breakdown:**
- {{ Component A }}: {{ X }}h ({{ Y }}%)
- {{ Component B }}: {{ Z }}h ({{ W }}%)
- ...

---

## 6. Acceptance Criteria
- [ ] {{ Criterion 1 }}
- [ ] {{ Criterion 2 }}

## 7. Risks & Unknowns
- **Assumptions:** {{ Critical assumptions made during planning }}
- **Gaps:** {{ Areas where context is missing or incomplete }}

```

</output_template>

## 4. Examples

### Example: Backend Task with Subtasks

**Task:** `Implement User Deletion Cascade`

**Plan (Section 4):**
```markdown
### Backend – User Deletion System
1. [ ] **Implement User Deletion Cascade (3 subtasks)**
   
   1.1. **Add CASCADE constraint to comments.user_id**
   - **Details:** Create migration `migrations/0042_add_cascade.sql` to add `ON DELETE CASCADE`.
   - **Files:** `migrations/0042_add_cascade.sql`, `db/schema.sql`

   1.2. **Update Service Logic**
   - **Details:** Update `CommentService.deleteByUserId` to handle post-deletion cleanup logging.
   - **Files:** `src/services/CommentService.ts`

   1.3. **Add integration test**
   - **Details:** Write test verifying comments vanish when user is deleted.
   - **Files:** `tests/integration/user-deletion.test.ts`
```

**Estimation (Section 5):**
| Component | Task | Estimate | Risk | Notes |
|-----------|------|----------|------|-------|
| Backend | Add CASCADE constraint | 2.5h | Low | DB Migration |
| Backend | Update Service Logic | 2.5h | Low | Cleanup logging |
| Backend | Add integration test | 2.5h | Low | Verification |
| **Total** | | **7.5h (100%)** | | **Confidence: High** |

**Breakdown:**
- Backend: 7.5h (100%)

### Anti-Pattern: Vague Tasks (Do NOT produce this)

| Component | Task | Estimate | Risk | Notes |
|-----------|------|----------|------|-------|
| Backend | Set up everything | 16h | Low | |
| Frontend | Build the UI | 20h | Medium | |

**Why this fails:** Tasks are not atomic, estimates exceed 8h without subtasks, no file paths or function names specified, risk ratings are unjustified.

