---
name: gitlab-mr-review
description: Review a GitLab Merge Request and provide findings, and post structured review comments with issue explanation plus pseudo code fixes. Use this skill when asked to review a Gitlab Merge request.
metadata:
    author: "Martin Roest <martin.roest@dawn.tech>"
    version: 1.3.1
---

# GitLab MR Review Workflow Skill

## Purpose

The purpose of this skill is to provide constructive and comprehensive feedback on code changes. The primary goals are:

- **Quality Assurance**: Identify bugs, potential logic errors, and edge cases.
- **Maintainability**: Ensure code is readable, modular, and consistent with the existing architecture.
- **Security**: Detect common security vulnerabilities and privacy risks. Validate against OWASP Top 10 where applicable.
- **Education**: Provide explanations and context for suggested changes to help the author grow.

This workflow is **read-first** and **non-invasive**:

- Do not modify repository files.
- Analyze MR content and discussions.
- Post comments only when explicitly requested.

## Inputs Required

1. **MR identifiers**: Either an MR URL or `{project_id, merge_request_iid}`
2. **Desired action** (to be confirmed in Step 6 report):
   - **Option 1**: Report only (show findings, no GitLab action).
   - **Option 2**: Post comments (publish inline draft notes).
   - **Option 3**: Post + Approve (comments + approval).
   - **Option 4**: Post + Request Changes (signal that changes are required).

## Required Tools

Before starting, verify the following tools are available. **If any tool is unavailable**, stop and ask the user to provide an MR export (`.patch` file or raw diff) to review offline.

| Tool | Purpose | Step |
|---|---|---|
| `mcp_gitlab_get_merge_request` | Fetch MR metadata (title, branches, diff refs) | Step 2 |
| `mcp_gitlab_mr_discussions` | Retrieve existing review threads | Step 2 |
| `mcp_gitlab_get_branch_diffs` | Retrieve diff between branches | Step 4 |
| `file_search` / `semantic_search` / `read_file` | Explore codebase conventions | Step 1 |
| `run_in_terminal` | Execute git worktree commands | Step 3, 7-E |
| `mcp_gitlab_create_draft_note` | Create inline review comments | Step 7-A |
| `mcp_gitlab_list_draft_notes` | Verify draft notes created | Step 7-A |
| `mcp_gitlab_update_draft_note` | Correct missing draft notes | Step 7-A |
| `mcp_gitlab_bulk_publish_draft_notes` | Publish all drafts at once | Step 7-A |
| `mcp_gitlab_approve_merge_request` | Approve the MR | Step 7-B |
| `mcp_gitlab_execute_graphql` | Request changes via GraphQL mutation | Step 7-C |
| `mcp_gitlab_update_merge_request` | Add current user as reviewer (if needed) | Step 7-C |

**Prohibited**: Do not use raw `curl` or `git` CLI commands for API interactions.

---

## Workflow

Follow these steps in order. Do not skip a step.

### Step 1 — Gather Codebase Context

Quickly identify the project's conventions and architecture:

1. Search for key structural files (framework config, `README`, linting configs).
2. Read project-level instructions (`.github/copilot-instructions.md`, `AGENTS.md`, `CLAUD.md` development docs).
3. Document: language, framework, architectural patterns, naming conventions, test strategy.
4. Store these observations as your **review baseline** for code analysis.

**Fallback if conventions are unclear**: Infer standards from the detected language/framework:

- **PHP**: PSR-12, PSR-4 (autoloading).
- **Python**: PEP 8, type hints (PEP 484).
- **JavaScript/TypeScript**: ESLint, Google/Airbnb style guides.
- **Go**: `gofmt`, idiomatic Go patterns.
- If still unclear, note this assumption in the final report and apply general best practices (SOLID, DRY, KISS).

---

### Step 2 — Fetch GitLab MR Details

Retrieve all MR metadata needed for the review.

1. Call `mcp_gitlab_get_merge_request` with `{project_id, merge_request_iid}`.
   - Extract: title, description, source/target branches, author, labels, milestone, diff refs (`base_sha`, `start_sha`, `head_sha`).
   - **Note diff refs**: These are required in Step 7-A for inline comment positioning.
2. Call `mcp_gitlab_mr_discussions` to load existing review threads.
   - Note all open and resolved threads to avoid posting duplicate feedback.
3. Note the MR description for stated intent, linked issues, and breaking-change flags.

---

### Step 3 — Checkout MR Branch

Create an isolated, read-only worktree for file access:

```bash
# Sanity check: prune orphaned worktrees from previous runs
git worktree prune

# Fetch the source branch to ensure it exists locally
git fetch origin {source_branch}:{source_branch} 2>/dev/null || true

# Create isolated worktree in detached state
REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_PATH="${REPO_ROOT}/.tmp/mr-review-{merge_request_iid}"
git worktree add --detach "${WORKTREE_PATH}" {source_branch}

# Lock worktree to prevent accidental modifications
git worktree lock "${WORKTREE_PATH}"
```

Store `worktree_path = "${REPO_ROOT}/.tmp/mr-review-{merge_request_iid}"` for Steps 4–5. **Read-only enforced** — the worktree is locked to prevent file modifications.

---

### Step 4 — Retrieve and Parse the Diff

1. Call `mcp_gitlab_get_branch_diffs` using the source and target branches.
2. Build a file-change inventory: list each changed file with its change type (added / modified / deleted).
3. Prioritise files for review:
   - **High priority**: core business logic, security-sensitive code, public APIs, data models.
   - **Lower priority**: generated files, lock files, migration snapshots, test fixtures.
4. *Constraint*: If the diff exceeds 20 files, focus on high-priority files first and note skipped files in the report.
   - **Option to user**: If >20 files are detected, ask the user whether they want a full review despite the scope; flag this in the report if high-priority files are skipped.

---

### Step 5 — Analyze & Classify Findings

Analyse each prioritised file against the review baseline. For each changed section, check:

- **Correctness** — logic errors, edge cases, incorrect conditionals.
- **Maintainability** — architecture adherence, DRY, SOLID, readability, clarity.
- **Security** — OWASP Top 10, injection risks, exposed secrets, PII.
- **Consistency** — coding standards, naming conventions, codebase patterns (from Step 1 baseline).
- **Scope / Description alignment** *(if MR description is available)* — validate that the actual diff matches the stated intent in the MR description:
  - Identify any changed files or logic that are **not mentioned** in the description (scope creep or unintended changes).
  - Identify any requirements stated in the description that appear to be **missing** from the diff.
  - Flag discrepancies as 🟡 **Request for Change** findings with an explanation of what was expected vs. what was found.
  - If the MR description is empty or absent, skip this check and note its absence in the report.

**Determine exact line numbers** using the worktree (`read_file` from `{worktree_path}/{file_path}`):

- For added/modified lines: extract `new_line` from the diff API hunk.
- For deleted lines: extract `old_line` from the diff API hunk.

**Record each issue:**

- File path and exact line number(s).
- Short description.
- Risk/impact.
- Suggested fix (pseudo code).

**Classify by severity:**

Use the following guidance table to categorize findings:

| Severity | Category | Examples | Action |
|----------|----------|----------|--------|
| 🔵 **Optional** | Style, naming, minor formatting | Whitespace inconsistency, code comments, doc formatting | Can be addressed in a follow-up or ignored |
| 🟡 **Request for Change** | Logic errors, architectural issues, configuration gaps | Incorrect conditionals, missing error handling, data consistency issues, missing translations, naming convention, typos in labels, unused imports, performance concerns, architectural inconsistency, breaking changes, API contract violations | Must be addressed before merge |
| 🔴 **Request for Change** | Security vulnerabilities, severe bugs | SQL injection, XSS vulnerability, exposed secrets/credentials, authentication bypass, buffer overflow, data loss risk, race conditions, privilege escalation | Must be addressed before merge |

**Note on Critical findings**: When a finding is a security vulnerability or severe bug, prefix the title with 🔴 (e.g., 🔴 **Request for Change — SQL injection vulnerability**) Always flag critical findings explicitly to the user before approval.

**Before compiling the report**: Every finding must use the **Comment Template structure**

- **READ FIRST**: "Comment Template" section below
- **Title** with severity and short topic
- **Relevant lines** — file path and line number(s)
- **Issue** — what is problematic
- **Why it matters** — risk/maintenance impact
- **Suggested fix** (pseudo code)

Do not invent alternative formats or omit any field.

Compile report grouped by severity with totals.

---

### Step 6 — Present Findings to the User

Present the full findings report to the user **before taking any action**.

The report must include:

- MR title, source → target branch, author.
- Finding totals per severity.
- Each finding formatted **exactly** using the **Comment Template structure** defined above.

**Pause here and await user confirmation.** Ask the user to choose one of the following actions:

> **What would you like me to do next?**
>
> 1. **Report only** — no GitLab action, findings are shown above.
> 2. **Post comments** — publish all findings on the MR.
> 3. **Post + Approve** — publish all findings and approve the MR.
> 4. **Post + Request Changes** — publish all findings on the MR (signals changes are required).

Do not call any posting, approval, or state-change tool until the user selects one of the options above.

---

### Step 7 — Execute Chosen Action (Only When Confirmed)

Based on the user's choice from Step 6, follow the matching sub-procedure below.

#### 7-A: Post Comments (options 2, 3, and 4)

Post all findings as **inline draft notes**, then publish them in a single batch.

1. **Create draft notes** — for each finding, call `mcp_gitlab_create_draft_note`:
   - Provide `project_id`, `merge_request_iid`.
   - Set `body` to the rendered Comment Template output from Step 7.
   - Provide a `position` object (inline diff comment):
     - `position_type: "text"` for code diffs (use `"file"` only if diff positioning fails).
     - `base_sha`, `head_sha`, `start_sha` from Step 2 diff refs.
     - **File path**: Use `new_path` for added/modified files; use `old_path` for deleted files.
     - **Line numbers from diff API**: Use `new_line` (for added/modified lines) or `old_line` (for deleted lines) as extracted and validated in Step 5. These MUST correspond to actual hunk boundaries from `mcp_gitlab_get_branch_diffs`.
   - Save each returned `draft_note_id` for verification.

2. **Positioning Fallbacks** (if inline positioning fails):
   - **Hunk mismatch or stale diff**: If the diff positioning is rejected by the API, the diff may be stale or the repository state has changed.
     - Retry with `position_type: "file"` on the relevant file path.
     - Include exact file path and line references inside the comment body using the Comment Template structure.
   - **Renamed or moved files**: For files that have been renamed/moved in the MR:
     - Use `new_path` for new sections; use `old_path` for removed sections.
     - If GitLab rejects the position, fallback to file-level comment.
   - **GitLab API error** (e.g., `line_code can't be blank`, invalid position object):
     - Log the error and file path.
     - Retry with `position_type: "file"` on the relevant file path.
     - Include exact file path and line references inside the comment body using the Comment Template structure.

3. **Proceed to draft verification**

4. **Proceed to draft verification** — verify all draft notes were created successfully.

5. **Verify drafts** — call `mcp_gitlab_list_draft_notes` with `{project_id, merge_request_iid}`.
   - Confirm the number of draft notes matches the number of findings.
   - Correct any missing notes using `mcp_gitlab_update_draft_note` before publishing.

6. **Publish** — call `mcp_gitlab_bulk_publish_draft_notes` with `{project_id, merge_request_iid}`.
   - All drafts become visible on the MR simultaneously.

#### 7-B: Approve (option 3 only)

After all draft notes are published (7-A complete):

1. Call `mcp_gitlab_approve_merge_request` with `{project_id, merge_request_iid}`.
2. Confirm approval was recorded (check response for approval state).
3. Only approve when **no 🔴 critical findings** were posted. If critical security or severe bug findings exist, warn the user and ask to confirm they still want to approve despite the issues.

#### 7-C: Request Changes (option 4 only)

After all draft notes are published (7-A complete):

1. **Validate reviewer assignment** — Before attempting the Request Changes mutation:
   - Retrieve the current MR state using `mcp_gitlab_get_merge_request` with `{project_id, merge_request_iid}`.
   - Extract the `reviewers` array from the response.
   - Check if the current user (typically identified by GitLab session context) is listed in the reviewers.
   - **If the current user is NOT a reviewer**: Call `mcp_gitlab_update_merge_request` to add yourself as a reviewer:
     ```
     Parameters:
     - project_id: the project identifier
     - merge_request_iid: the MR IID
     - reviewer_ids: array containing the ID of the current user (retrieve from GitLab user context)
     ```
   - Confirm the update was successful before proceeding to step 2.

2. **Submit a formal "Request Changes" review** using the GitLab GraphQL mutation `mergeRequestRequestChanges`:

   ```graphql
   mutation RequestChanges($projectPath: ID!, $iid: String!) {
     mergeRequestRequestChanges(input: { projectPath: $projectPath, iid: $iid }) {
       mergeRequest {
         iid
         title
         reviewers {
           nodes {
             username
             mergeRequestInteraction {
               reviewState
             }
           }
         }
       }
       errors
     }
   }
   ```

   Call `mcp_gitlab_execute_graphql` with the query above and variables:
   - `projectPath`: the full project path (e.g. `"group/project"`) — derived from the project identifier used in Step 2.
   - `iid`: the merge request IID as a string.

3. **Verify the response**:
   - Confirm `errors` is empty.
   - Check that the current user's `reviewState` in the response is `REQUESTED_CHANGES`.
   - If the mutation returns errors (e.g., GitLab API issue), inform the user with the error details — the published inline comments still communicate the required changes.

4. Include summary note to inform the user:
   - Include a summary with key findings and post it as an overall comment on the MR.

#### 7-D: Report Only (option 1)

If the user chose **"Report only"** (no GitLab action):

1. Skip Steps 7-A, 7-B, and 7-C.
2. Proceed directly to 7-E cleanup (worktree removal below).
3. Summary to user: Provide the findings report and note that no draft notes were posted.

#### 7-E: Clean Up

After all actions are complete (or if option 1 was chosen):

1. **Remove the git worktree with verification**:

   ```bash
   REPO_ROOT=$(git rev-parse --show-toplevel)
   WORKTREE_PATH="${REPO_ROOT}/.tmp/mr-review-{merge_request_iid}"
   
   # Unlock before removal
   git worktree unlock "${WORKTREE_PATH}" 2>/dev/null || true
   
   # Attempt removal with error handling
   if git worktree remove "${WORKTREE_PATH}"; then
     echo "Worktree cleaned up successfully."
   else
     echo "WARNING: Failed to remove worktree at ${WORKTREE_PATH}"
     echo "Run 'git worktree prune' manually to clean orphaned entries."
   fi
   ```

2. **Report back** to the user:
   - **For Report only (option 1)**: Confirm findings were presented; no actions taken.
   - **For Posted comments/approval/request-changes**: Draft note IDs that were published, approval state (if applicable), MR state change (if applicable) and include summary with key findings.
   - If any fallback path was used, explain why in one sentence.
   - If worktree cleanup failed, notify the user to run `git worktree prune` manually.

---

## Comment Template Example

Refer to the **Comment Template structure** defined in Step 5. Here is a concrete example:

```text
🟡 Request for Change — avoid repeated magic string

Relevant lines
- path/file.twig around line 108
- path/partial.twig around line 99

Issue
- Repeated literal condition in multiple files.

Why this matters
- Rename risk and inconsistent behavior.

Suggested fix (pseudo code)
{# parent #}
set boolean once
pass boolean to include

{# child #}
branch on boolean
```

## Output Style for User

- Keep summary concise.
- State exactly what was posted (thread IDs/note IDs when available).
- If a fallback path was used, explain why in one sentence.

## Guardrails

- Post comments only when user selects option 2, 3, or 4 at the end of Step 6 (do not post for option 1).
- Only approve (option 3) if **no 🔴 critical findings** — otherwise ask user re-confirmation.
- Option 4 (request changes) relies on published review comments; no additional MR state change is required.
- Never merge, alter code, or use alternate providers (GitHub, Jira, etc.).
- Keep findings tied to concrete diff evidence from the branch worktree.
- If the workflow is interrupted (user cancels, agent crashes), manually run `git worktree prune` to clean orphaned entries and recover disk space.

## Completion Checklist

- [ ] **Step 1**: Codebase context gathered, conventions documented
- [ ] **Step 2**: MR metadata fetched (title, branches, diffs refs, author), discussions loaded
- [ ] **Step 3**: Worktree created at `.tmp/mr-review-{merge_request_iid}`
- [ ] **Step 4**: Diff parsed, files prioritised, high/low priority files identified
- [ ] **Step 5**: Code analysed, findings classified by severity, description alignment validated (if available), Comment Template fields populated
- [ ] **Step 6**: Report presented to user, action confirmed (options 1–4 selected)
- [ ] **Step 7**: Appropriate sub-procedure executed (7-A/B/C/D/E), worktree removed, summary reported
