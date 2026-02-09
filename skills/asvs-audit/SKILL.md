---
name: asvs-audit
metadata:
    author: "Martin Roest <martin.roest@dawn.tech>"
    version: 2.1.0
    asvs-version: 5.0.0
description: OWASP ASVS 5.0 Level 1 security audit with deterministic, evidence-based findings. Use this when asked for a security audit or asvs audit.
---

# OWASP ASVS 5.0 Level 1 Security Audit

**Role**: You are an Application Security Expert. Conduct systematic, evidence-based security audits against OWASP ASVS 5.0 Level 1 requirements using the bundled CSV as the canonical source.

## 🎯 Audit Scope & Constraints

### What You Will Do
1. Profile the target codebase technology stack
2. Systematically verify all 70 ASVS Level 1 requirements from the CSV
3. Classify each requirement as: ✅ PASS | ⚪ N/A | ⚠️ NEEDS_REVIEW | ❌ FAIL → Severity
4. For each finding, provide exact code location + evidence + remediation
5. Generate a deterministic audit report file based on a skeleton template

### What You Will NOT Do
- Use a different ASVS requirement list
- Provide generic remediation without code examples
- Mark PASS without specific evidence
- Create your own markdown report structure
- Capture API keys, secrets, or PII in evidence sections

## 📋 Prerequisites

**Tools Required**: Git (optional), File search, Grep, Terminal  
**Access Required**: Full read access to target repository  
**Inputs Required**: Target repo path, project name (derived from package.json/pyproject.toml/git repo name)  
**CSV Location**: `./assets/OWASP_Application_Security_Verification_Standard_5.0.0_L1_en.csv` (skill workspace)

## 🚨 Critical Rules (Non-Negotiable)

1. **CSV is canonical**: Use exact row order from CSV (items 1-70), never sort or reorder. `internal_num` = 1-based CSV row number (1–70), displayed as `#N` in all outputs.
2. **Complete coverage**: All 70 items must be evaluated—no skipping
3. **Evidence required**: Every item needs concrete proof (file:line, config value, framework feature, or N/A reason)
4. **No truncation**: Include full `req_description` text from CSV in all outputs
5. **Deterministic evaluation**: Use the Decision Tree for every requirement
6. **Single write at end**: Build complete report in memory during Phase 3, write file once using `create_file`

## Exclusions

Skip these directories and files during analysis (they contain third-party or generated code):

- `node_modules/`, `vendor/`, `packages/` (dependency directories)
- `dist/`, `build/`, `out/`, `target/`, `.next/` (build outputs)
- `.git/`, `.svn/`, `.hg/` (version control)
- `*.min.js`, `*.bundle.js` (minified/bundled files)
- `coverage/`, `.nyc_output/` (test coverage)
- `__pycache__/`, `*.pyc`, `.pytest_cache/` (Python cache)
- Test files: `*.test.*`, `*.spec.*`, `*_test.*`, `test_*.*`, `__tests__/`, `tests/`, `spec/` (test code)

**Lock files** (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `poetry.lock`): Exclude from general searches. Permit targeted reads only during V10 (Malicious Code / Dependencies) evaluation.
- **🔒 Sensitive files** (do not read): `.env`, `.env.*`, `secrets.json`, `credentials.json`, `*.pem`, `*.key`, `*.pub`, AWS credentials files

---

## How to Evaluate Requirements

For each of the 70 ASVS items, collect evidence using the **Decision Tree** (see section below) and classify as: ✅ PASS | ⚪ N/A | ⚠️ NEEDS_REVIEW | ❌ FAIL.

**Evidence must be concrete and specific**:

Evidence MUST follow the strict formats defined in [`references/evidence-patterns.md`](./references/evidence-patterns.md). Do not use free-form text for evidence.

---

## Decision Tree (Integrated Evaluation & Search)

Apply this 4-step process to **every** requirement. **STOP** searching immediately once you reach a definitive conclusion (Lazy Verification).

**Step 1: Determine Applicability (Shallow Search)**
*Source: `package.json`, file extensions, tech stack profile, top-level configs.*

1.  **Is this irrelevant to the tech stack?** (e.g., Java requirements in a Node.js app)
    -   **YES** → 🛑 **STOP**. Mark **⚪ N/A** (Evidence: "Tech stack is X, not Y").
    -   **NO** → Continue.
2.  **Does the feature exist?** (e.g., searching "upload", "websocket", "sql" globally)
    -   **NO** (Zero results found) → 🛑 **STOP**. Mark **⚪ N/A** (Evidence: "Feature X not used in codebase").
    -   **UNCERTAIN** or **YES** → Proceed to Step 1.5.

**Step 2: Check Framework Defaults (Short-Circuit)**
*Source: [`references/framework-defaults.md`](./references/framework-defaults.md) table, matched against detected framework from Phase 1.*

1.  **Is this requirement covered by a known framework default?** - Match the requirement's ASVS chapter against the framework table.
    -   **YES** → Search for the corresponding **bypass pattern** from the table.
        -   **Bypass found** → Mark **❌ FAIL** with bypass location. Proceed to Step 3.
        -   **No bypass found** → 🛑 **STOP**. Mark **✅ PASS** (Evidence: `framework:<name>:<feature>`).
    -   **NO** → Proceed to Step 2.

**Step 3: Verify Controls (Deep Search)**
*Source: Specific source files, middleware chain, security configuration, library definitions.*

1.  **Is the control implemented?** (Search for specific libraries, config flags, or code patterns)
    -   **YES** → 🛑 **STOP**. Mark **✅ PASS** (Evidence: `file:line` or config value).
2.  **Is the control missing or bypassed?**
    -   **YES** → Mark **❌ FAIL**. Proceed to Step 3.
3.  **Is verification impossible?** (Binary file, obfuscated, or requires external server access)
    -   **YES** → 🛑 **STOP**. Mark **⚠️ NEEDS_REVIEW** (Reason: "Cannot analyze binary/external system").

**Step 4: Assign Severity (Failures Only)**
*Read full Source: [`references/severity-guidance.md`](./references/severity-guidance.md)*

1.  Determine severity based on the ASVS Chapter (e.g., Auth = High).
2.  **Override** only if you find specific code proving lower/higher impact.
3.  **Output**: Must include location of failure (`file:line`) or the search query that yielded zero results (`missing:validated_jwt_middleware`).

---

> **Important (CSV location)**: The ASVS Level 1 CSV is bundled with this *skill* itself, not the target application repo being audited.
> It is located in this skill’s own `assets/` directory (a sibling of this `SKILL.md`).
>
> **Path Resolution**:
>
> - **Skill workspace path**: The directory containing this `SKILL.md` file (e.g., `/home/username/.copilot/skills/asvs-audit/`)
> - **CSV absolute path**: `{skill_workspace}/assets/OWASP_Application_Security_Verification_Standard_5.0.0_L1_en.csv`
> - When using tools, always resolve to the **absolute path** to avoid ambiguity
>
> **Context Switching**: This audit operates across TWO directories:
>
> 1. **Skill workspace**: Where this `SKILL.md` and `assets/` folder reside (for reading the CSV)
> 2. **Target repo**: The application being audited (for all code analysis and git commands)
>
> When running git commands or analyzing code, always explicitly reference or `cd` to the **target repo path** first.
> When loading the ASVS CSV, use the **absolute skill workspace path**.

---

## Execution Flow: 3 Phases

The audit is divided into three sequential phases.

### Phase 1: Prepare

1. **Context Gathering** (Performance optimization—done once):
   - Tech Stack Profile: Language, framework, database, key libraries
   - Dependency Lists: Read `package.json`, `requirements.txt`, etc. once; store in memory
   - Framework Defaults: Load framework-specific security features from [`references/framework-defaults.md`](./references/framework-defaults.md)
   - Git Metadata: Run `git rev-parse --short HEAD` once; extract app version from manifest
   - Directory Structure: Top-level directories, component types (respect exclusions)

2. **Monorepo Detection**: If the target repo contains multiple services/packages, prompt the user to specify the target service. Default to the repository root if no guidance is given. Prefix all evidence with `[component]` per `references/evidence-patterns.md`.

3. **Load CSV**:
   - File: `./assets/OWASP_Application_Security_Verification_Standard_5.0.0_L1_en.csv` (in skill workspace)
   - Columns: `chapter_id`, `chapter_name`, `section_id`, `section_name`, `req_id`, `req_description`, `L`

### Phase 2: Evaluate (Chapter by Chapter)

Process chapters sequentially (V1 → ... → V14) to manage context window limits.

1.  **Load Scope**: Extract requirements for the current chapter only.
2.  **Execute Decision Tree**:
    -   Apply the **Step 1 → Step 2** logic for every requirement.
    -   **Parallelization**: Within a chapter, batch independent grep/search operations for multiple requirements into parallel tool calls when no data dependency exists between them.
    -   **Efficiency Constraint**: Use `grep` first. Only use `read_file` if grep indicates a match.
    -   **Large File Handling**: If a targeted file is large (>500 lines), read only the **first 500** and **last 100** lines initially. Mark evidence as "(Sampled)".
3.  **Accumulate & Flush**:
    -   Save findings to your internal list.
    -   Do not re-read files from previous chapters. Carry forward only the Tech Stack Profile and Dependency List. Begin each chapter with fresh searches against the target codebase.

### Phase 3: Generate (Single Write)

Once all 70 items are evaluated:

1. **Calculate Statistics**: Count PASS/FAIL/N/A/NEEDS_REVIEW; compute compliance score = PASS / (70 - N/A - NEEDS_REVIEW) × 100

2. **Build Complete Report in Memory** (using `references/REPORT-TEMPLATE.md` as skeleton):
   - **Essentials rules**: Adhere strictly to the template. Do not alter headings, sections, or layout.
   - **Header sections**: report details, app details, tech stack, introduction
   - **Findings section**: All FAIL items (in CSV order), exact req_description, evidence, remediation. `{{ internal_num }}` is the 1-based CSV row index.
   - **Verification table**: Exactly 70 rows (Items 1–70, CSV order), no duplicates. `{{ internal_num }}` is the 1-based CSV row index.
   - **Summary section**: Statistics, compliance score, review debt
   - **Conclusion section**: Overall posture, key risks, recommended next steps

3. **Validate Before Writing**:
   - ✅ All 70 items present in verification table (no gaps, no duplicates)
   - ✅ All FAIL items present in findings section (same order as CSV)
   - ✅ Zero unreplaced `{{ placeholders }}`
   - ✅ Zero `<!-- ... -->` comments
   - ✅ Every FAIL item has evidence location (file:line or missing:what_was_searched)
   - ✅ **🔒 No secrets, API keys, PII, or unredacted sensitive values in report**
   - ✅ File name format: `{project_name}-ASVS-L1-audit-YYYY-MM-DD.md`

4. **Write Entire Report** to file in **one operation** using `create_file` tool

5. **Confirm Completion**:
   - Output Coverage Statistics from the report to chat
   - Confirm filename and path where report was saved

---

## Error Handling

| Scenario | Action |
|----------|--------|
| CSV file missing/corrupted | STOP audit, report error: "ASVS CSV not found at expected path" |
| Target codebase empty | STOP audit, report: "No source files found in target repository" |
| Target codebase inaccessible | STOP audit, report: "Cannot access target path: [path]" |
| Git commands fail | Set Git Commit to `unknown`, continue audit |
| Tool fails mid-audit | Mark as **⚠️ NEEDS_REVIEW** with note: "Verification failed due to tooling error — manual review required". |
| Token/context limit approaching | Complete current chapter, save partial report with `[PARTIAL]` prefix, note last completed item |
| File too large to read | Sample first 500 lines + last 100 lines, note in Evidence: "Large file - sampled" |

---

## Examples

For detailed examples of report formatting, finding documentation, and evidence patterns, see [EXAMPLES.md](./EXAMPLES.md).
