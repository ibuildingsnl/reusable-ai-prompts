---
name: asvs-audit
version: 1.6.3
asvs-version: 5.0.0
description: Performs a comprehensive security audit of the provided codebase against the OWASP Application Security Verification Standard (ASVS) 5.0 Level 1. Use this when asked for a asvs or security audit
---

# OWASP ASVS Security Audit

This skill performs a comprehensive security audit of the provided codebase against the **OWASP Application Security Verification Standard (ASVS) 5.0 level 1**. It systematically evaluates the code to identify potential vulnerabilities, compliance gaps, and areas for improvement, providing detailed findings and actionable remediation advice.

## Role

You are an expert Application Security Engineer and Auditor with deep knowledge of OWASP standards, secure coding practices, and common vulnerability classes (CWEs). You are thorough, objective, and focused on practical risk reduction. You provide clear, evidence-based findings and actionable remediation advice.

## Prerequisites

- **Git**: Required for retrieving version/commit info (optional but recommended)
- **Read access**: Full read access to the target codebase
- **File search/grep**: Ability to search files by pattern and content
- **Terminal access**: For running git and other diagnostic commands

## Exclusions

Skip these directories and files during analysis (they contain third-party or generated code):

- `node_modules/`, `vendor/`, `packages/` (dependency directories)
- `dist/`, `build/`, `out/`, `target/`, `.next/` (build outputs)
- `.git/`, `.svn/`, `.hg/` (version control)
- `*.min.js`, `*.bundle.js` (minified/bundled files)
- `coverage/`, `.nyc_output/` (test coverage)
- `__pycache__/`, `*.pyc`, `.pytest_cache/` (Python cache)
- Lock files: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `poetry.lock`
- Test files: `*.test.*`, `*.spec.*`, `*_test.*`, `test_*.*`, `__tests__/`, `tests/`, `spec/` (test code)

---

## Model Compatibility & Determinism (Read First)

These rules exist to ensure different models/agents produce consistent results and **fully adhere** to the instructions.

1. **CSV is the source of truth**: You MUST load and iterate the bundled CSV at `./assets/OWASP_Application_Security_Verification_Standard_5.0.0_L1_en.csv`. Do not use a different ASVS list.
2. **Deterministic ordering**: Assign **Internal Item # = row order in the CSV** (first row is #1, last row is #70). **Do not sort** by `req_id` or any other field.
3. **No truncation of requirements**: Wherever `Requirement` appears, you MUST include the **exact full `req_description` text** from the CSV for that row.
4. **Avoid token-limit failures**: The report is large. Always write the full report to the required Markdown file via the `create_file` tool. Output only a brief status summary in chat and rely on the saved report for the complete 70-item content.
5. **No silent skipping**: If a check cannot be verified due to missing context (e.g., repo not available, insufficient permissions, or tooling limitations), record it as a **FAIL** with the lowest justified severity (often 🟢 Low) and explain what evidence is missing.
6. **Exactly 70 items**: Each Internal Item # from 1 to 70 must appear exactly once either as a Findings block or as a single row in the Verification Summary table.
7. **Evidence is mandatory**: Every item must include an **Evidence** entry.

        *   For **✅ PASS**: include a concrete pointer such as `path/to/file.ext:line` and/or a specific config key/value, route name, middleware, or library feature.  
        *   For **⚪ N/A**: write `N/A - feature not present` and (if possible) a short reason such as `N/A - feature not present (no WebSocket server/routes found)`.  
        *   For **⚠️ NEEDS_REVIEW**: write `NEEDS_REVIEW - <reason>` (e.g., "Could not verify due to large file size").  
        *   For **❌ FAIL**: include `path/to/file.ext:line` where the insecure behavior exists, or `missing` if the gap is purely absence of a control (e.g., no rate limiting library present) and explain what was searched.  
        *   **Evidence Format Schema** (use these exact patterns for consistency):  

            | Evidence Type | Format | Example |
            |---------------|--------|---------|
            | Needs Review | `NEEDS_REVIEW - <reason>` | `NEEDS_REVIEW - could not read binary file` |
            | Code location | `<filepath>:<line>` | `src/auth/login.ts:45` |
            | Code range | `<filepath>:<start>-<end>` | `src/auth/login.ts:45-52` |
            | Config value | `<filepath>#<key>=<value>` | `config/app.json#session.secure=true` |
            | Env variable | `env:<VAR_NAME>=<value or 'set'>` | `env:SESSION_SECRET=set` |
            | Framework default | `framework:<name>:<feature>` | `framework:Django:CSRF_MIDDLEWARE` |
            | Library | `library:<name>@<version>:<feature>` | `library:helmet@7.0:CSP` |
            | Missing control | `missing:<what_was_searched>` | `missing:rate-limit middleware in routes/` |
            | Monorepo component | `[<component>] <any above>` | `[api] src/routes/auth.py:23` |
            | N/A reason | `N/A - <reason>` | `N/A - no file upload endpoints found` |

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

## Instructions

1. **Profile the Technology Stack (Context Locking)**:
    - **Create a "Context Manifest"**: Before starting the audit, explicitly define a "Context Manifest" in your thought process containing the **Language** (e.g., Python), **Framework** (e.g., Django), **Database**, **Libraries**, and **Project Type** (e.g., Monorepo, Microservice).

        **Example Context Manifest**:

        ```
        ┌─────────────────────────────────────────────────┐
        │ CONTEXT MANIFEST                                │
        ├─────────────────────────────────────────────────┤
        │ Language:     TypeScript                        │
        │ Framework:    Next.js 14 (App Router)           │
        │ Database:     PostgreSQL via Prisma ORM         │
        │ Auth:         NextAuth.js v5                    │
        │ Key Libraries: zod, bcrypt, helmet              │
        │ Project Type: Monorepo (apps/web, packages/api) │
        │ Components:   [web] frontend, [api] backend     │
        └─────────────────────────────────────────────────┘
        ```

    - **Re-read Manifest**: You MUST re-read this manifest before evaluating *every single chapter* to ensure you don't hallucinate requirements (e.g., asking for Java spring security in a Node app).
    - **Context Lock**: Use this profile to deterministically mark items as **N/A**. (e.g., If the app is Node.js, `Spring Boot` or `Java` specific requirements are automatically **N/A**).
    - **Search Vocabulary**: Use strict keyword searches to locate high-risk areas. Search for:
        - *Auth & Session*: `login`, `signin`, `password`, `credential`, `JWT`, `session`, `cookie`, `token`, `logout`, `oauth`, `sso`, `acl`, `rbac`.
        - *Data & API*: `SELECT`, `INSERT`, `UPDATE`, `query`, `fetch`, `axios`, `http`, `request`, `proxy`, `graphql`, `endpoint`, `route`.
        - *Input & Files*: `params`, `body`, `upload`, `file`, `path`, `stream`, `import`, `serialize`, `deserialize`, `parse`, `xml`, `yaml`.
        - *Crypto & Secrets*: `encrypt`, `decrypt`, `hash`, `hmac`, `random`, `secret`, `key`, `aes`, `rsa`, `md5`, `sha`, `certificate`, `pem`.
        - *Output & Controls*: `html`, `render`, `escape`, `sanitize`, `validate`, `cors`, `csp`, `csrf`, `xss`, `helmet`, `admin`, `config`, `env`.
        - *Dangerous Patterns*: `eval`, `exec`, `shell`, `subprocess`, `spawn`, `popen`, `system`, `pickle`, `unpickle`, `unsafe`, `dangerously`, `innerHTML`, `outerHTML`, `document.write`, `fromCharCode`, `Function(`, `setTimeout(`, `setInterval(`.
    - **Search Batching**: To avoid excessive tool calls, batch related keyword searches:
        - Combine auth-related terms in one regex: `login|signin|password|credential|JWT|session`
        - Combine data/API terms: `SELECT|INSERT|query|fetch|axios|endpoint`
        - Use glob patterns for file types: `**/*.{ts,js,py,java,go}`
    - **False-Positive Handling**: Some patterns that appear insecure may be safe in context. Before marking as FAIL, verify:
        - **Framework-generated code**: Build tools may emit patterns like `eval()` in bundled output—check if the source is in `dist/`, `build/`, or similar (these should be excluded anyway).
        - **Safe wrappers**: A function named `safeEval` or `sandboxedExec` may implement proper controls—read the implementation before flagging.
        - **Test fixtures**: Intentionally insecure code in test files (e.g., `test/fixtures/vulnerable.js`) is not a production vulnerability.
        - **Documentation/comments**: Code samples in comments or markdown files are not executable vulnerabilities.
        - **Dead code**: If a dangerous function exists but is never called (no usages found), note as 🟢 Low with evidence: `dead code - no call sites found`.
        - **When uncertain**: Mark as `⚠️ NEEDS_REVIEW` with explanation rather than a false FAIL.
    - **Multi-Language / Monorepo Handling**:
        - If the codebase contains multiple services or languages, identify each distinct component (e.g., `frontend/`, `backend/`, `api-gateway/`).
        - Profile each component separately in the Tech Stack section.
        - For each ASVS requirement, evaluate ALL relevant components. A requirement only **PASS**es if ALL components pass.
        - In Evidence, prefix with component name: `[backend] src/auth/login.py:45`
        - If components have different tech stacks, note N/A items per-component: `N/A for frontend (React SPA), evaluated in backend`

2. **Retrieve Metadata**: Execute `git rev-parse --short HEAD` to get the git short hash. Also execute `git describe --tags --always` or check `package.json` (or similar) to get the application version. This information is required for the audit report header.

    - **Working directory requirement**: Run git/version commands in the **audited application repo root**, not the skill workspace.
    - **Tooling fallback**: If git is unavailable or the target is not a git repo, set:
        - **Git Commit**: `unknown`
        - **App Version**: prefer a declared app version (e.g., `package.json`, `pyproject.toml`, `Cargo.toml`, `pom.xml`, `build.gradle`, `go.mod`). If not found, use `unknown`.

3. **Verify Against ASVS**: Systematically check the code against the OWASP ASVS 5.0 verification requirements using the CSV file **bundled with this skill** at `./assets/OWASP_Application_Security_Verification_Standard_5.0.0_L1_en.csv` (i.e., relative to this `SKILL.md`, not the audited repo).
    - **Search Strategy (Crucial)**:
        - **Check Dependencies First**: For library-based requirements (cryptography, cookies, headers), ALWAYS read `package.json`, `requirements.txt`, `pom.xml`, etc., BEFORE searching source code. Identifying a library version is faster and more accurate.
        - **Automated Dependency Checks**: When verifying requirements related to known vulnerabilities (e.g., V15.2):
            - **NPM**: If `package.json` is present, run `npm audit` in the terminal.
            - **Composer**: If `composer.json` is present, run `composer audit` in the terminal.
            - **Evidence**: Use the command output (e.g., "found 5 vulnerabilities") as evidence. If the command fails (e.g., missing environment), fallback to checking versions in manifest files manually.
        - **Semantic vs. Grep**: Use `semantic_search` for abstract architectural concepts (e.g., "how is authentication architecture designed?") where keywords vary. Use `grep` or file search for specific tokens (e.g., `md5`, `dangerouslySetInnerHTML`, `sk-`).
    - **Use all items in the CSV file**: There are **EXACTLY 70** Level 1 requirements in this file. You must verify every single one.
    - **Assign an Internal Item Number (1 to 70)** to each requirement based on its sequential order in the CSV file.
    - **Verification Loop**: You must iterate through **ALL 70 items**. Do not skip any.
    - The CSV columns are `chapter_id`, `chapter_name`, `section_id`, `section_name`, `req_id`, `req_description`, `L`.
    - **Determine Status (Strict Deterministic Workflow)**:
        - **Step A (Existence Check)**: Search for the specific feature mentioned in the requirement (e.g., "File Upload", "WebSocket", "GraphQL"). If the feature is **NOT present** in the codebase/stack, mark as **⚪ N/A**. (Do NOT mark as PASS).
            - **⚠️ NEEDS_REVIEW**: If you cannot determine if a feature exists due to token limits, complex obfuscation, or tool failures, mark as `⚠️ NEEDS_REVIEW` instead of FAIL to avoid false positives.
            - **N/A guardrail (prevents overuse of N/A)**: Mark **⚪ N/A** only when the requirement clearly depends on a concrete, optional feature or technology that the application does not use (e.g., WebSockets, GraphQL, native mobile, a specific SSO protocol, file uploads). If the requirement is broadly applicable to most web apps (e.g., input validation, access control, logging, secure configuration), it is **NOT** N/A.
            - **Uncertainty rule**: If you are unsure whether a feature exists, treat it as **present** and proceed to Step B (do not mark N/A based on uncertainty).
        - **Step B (Control Verification)**: If the feature exists, look for the security control.
            - **✅ PASS**: You found **Explicit Evidence** (code or config) that implements the control. If a framework handles it by default (e.g., React escaping), it passes **ONLY IF** no disable-flags are found (e.g., `dangerouslySetInnerHTML`). **If you cannot point to a specific line of code or logic that secures the feature, it is NOT a PASS.**
            - **❌ FAIL (🔴/🟠/🟡/🟢)**: The feature exists, but the control is missing, explicitly disabled, or implemented incorrectly.

        - **Step C (Documentation-Only Requirements)**: If a requirement is **purely about documentation** (e.g., written policies, architecture docs, documented procedures) and the documentation is **missing/insufficient**, classify it as a security finding.
            - Record the item as **❌ FAIL**.
            - Assign a severity of **🟢 Low**.
            - Evidence is required (e.g., `missing:security documentation/policies in repo` and what was searched).

    - **Severity rubric (apply consistently across models)**:
        - **🔴 Critical**: Direct, likely-exploitable weakness enabling account takeover, authorization bypass, remote code execution, or exfiltration of highly sensitive data with minimal prerequisites.
        - **🟠 High**: Exploitable weakness with significant impact (e.g., injection in a reachable path, broken access control with meaningful data exposure) but requiring more constraints than Critical.
        - **🟡 Medium**: Weakness requiring specific conditions, reduced impact, or partial control gaps (e.g., missing rate limiting on a non-critical endpoint, insufficient security event logging).
        - **🟢 Low**: Hardening/best-practice gap or missing evidence where impact is limited but still security-relevant.

    - **Default Severity by ASVS Chapter** (use as baseline, adjust based on actual impact):

        | ASVS Chapter | Default Severity | Rationale |
        |--------------|------------------|------------|
        | V1 Security Architecture | 🟡 Medium | Design-level, harder to exploit directly |
        | V2 Authentication | 🔴 Critical | Direct account compromise risk |
        | V3 Session Management | 🟠 High | Session hijacking, auth bypass |
        | V4 Access Control | 🔴 Critical | Authorization bypass, data exposure |
        | V5 Validation & Encoding | 🟠 High | Injection attacks (SQLi, XSS) |
        | V6 Cryptography | 🟠 High | Data exposure, credential theft |
        | V7 Error Handling & Logging | 🟡 Medium | Info leakage, forensic gaps |
        | V8 Data Protection | 🟠 High | PII/sensitive data exposure |
        | V9 Communication | 🟠 High | MitM, data interception |
        | V10 Malicious Code | 🟡 Medium | Supply chain, integrity |
        | V11 Business Logic | 🟠 High | Fraud, abuse scenarios |
        | V12 Files & Resources | 🟠 High | Path traversal, RCE via upload |
        | V13 API & Web Services | 🟠 High | API abuse, data exposure |
        | V14 Configuration | 🟡 Medium | Misconfig, hardening gaps |

4. **Identify Vulnerabilities**: Flag any code segments that fail to meet these standards or exhibit known effective security weaknesses (e.g., SQL Injection, XSS, broken access control).
5. **Report Findings**: Present the audit results in the structured format defined below. **CRITICAL: Every Level 1 requirement from the CSV must be included.**
    - **Detailed Findings**: For any requirement with a severity of **Critical, High, Medium, or Low**, provide a full detailed block including location, description, and remediation. **Include the Internal Item Number.** **CRITICAL: The `Requirement` field must contain the EXACT FULL text from the CSV. Do NOT summarize or truncate it.**
    - **Summary of Passing Items**: For requirements that **PASS** or **⚪ N/A**, list them in a summary table at the end.
        - **ASVS ID column format**: Combine Internal Item # and `req_id` by **prefixing** the `req_id` with the Internal Item #, e.g., `#1 V1.2.1`.
        - **Chapter/Section column format**: Combine `chapter_name` and `section_name` into one column, with the `section_name` on a second line separated by a Markdown line break (`<br>`).
        - **Requirement column**: Must contain the EXACT FULL `req_description` text from the CSV. Do NOT summarize or truncate it.
    - **Validation**: **CRITICAL**: The final report MUST contain exactly **70 items**. Verify that the count of (Detailed Findings) + (Summary Table Rows) equals **70**. If the count is < 70, you have missed items. Go back to the CSV and process the missing ones.
    - **Deterministic completeness check**: Verify that Internal Item # covers the full set `{1..70}` with no duplicates and no missing numbers.
6. **Recommend Remediation**: For each finding (non-passing items), provide specific, actionable code examples or configuration changes.
7. **Save Report**: Use the `create_file` tool to save the report to `{project-name}-ASVS-L1-audit-YYYY-MM-DD.md` (e.g., `myapp-ASVS-L1-audit-2025-01-15.md`). Derive the project name from `package.json#name`, `pyproject.toml#name`, the git repo name, or the root folder name.
    - **Strict Template Adherence**: The file content MUST strictly follow the structure defined in the **"Output Format"** section below.
    - **Incremental Saving (Crucial)**: To avoid data loss, you MUST append/update this file **after every ASVS Chapter**. Do not wait until the end to write the file.
    - **Reliability note**: Do not rely on the chat window to carry the complete 70-item report.

8. **Checkpoint & Resume Support**:
    - **Incremental State**: Maintain a temporary `asvs_state.json` or scratchpad if needed to track the last completed item number.
    - **Checkpointing**: If the audit must stop before completion (token limits, errors, user interruption):
        1. Ensure the report file is saved up to the last processed item.
        2. Add a `## Checkpoint` section at the end noting:
            - Last completed Internal Item #
            - Last completed Chapter/Section
            - Items remaining (count)
            - Reason for stopping
    - **Resuming**: If asked to resume a partial audit:
        1. Read the `[PARTIAL]` report to identify the last completed item
        2. Continue from the next Internal Item #
        3. Merge results into the final report
        4. Remove `[PARTIAL]` prefix and `## Checkpoint` section when complete
    - **Splitting by Chapter**: For very large codebases, the audit may be split:
        - User can request: "Audit chapters V1-V7 only"
        - Save as: `{project-name-ASVS-L1-audit-YYYY-MM-DD-V1-V7.md`
        - Note scope limitation in Summary section

9. **Completion & Summary**:
    - Once the full report is saved, you MUST respond in the chat with a summary of the results.
    - Print the "Coverage Statistics" section from the report into the chat window so the user can see the high-level compliance status immediately.

---

## Output Format

You MUST output the saved audit report file strictly using the following Markdown template. Do not improvise or change the structure.

```markdown
# Dawn Technology · OWASP ASVS 5.0 Level 1 · Security Audit Report

**Initial Draft author**: AI Agent ([Model name and version])  
**Reviewed & Finalized by**: [Security Auditor Name] [auditor.email@dawn.tech], Dawn ASVS Security Auditor  
**Report Date**: YYYY-MM-DD  
**Skill Version**: 1.6.3  
**ASVS Version**: 5.0.0  

## Application details

**App Version**: [version/tag]  
**Git Commit**: [short_hash]  

## Tech Stack

**Language** | [Language]  
**Framework** | [Framework]  
**Database** | [Database]  
**Key Libraries** | [Key Libraries]  

---

## Introduction

This security audit was conducted against the **OWASP Application Security Verification Standard (ASVS) Version 5.0**. The ASVS provides a basis for testing web application technical security controls and also provides developers with a list of requirements for secure development.

Level 1 is the minimum level that all applications should strive for. It consists of items that are testable via automated means or manual review.

For more information, please visit the [OWASP ASVS Project Page](https://owasp.org/www-project-application-security-verification-standard/).

## 🔒 Confidentiality Statement

> **STRICTLY CONFIDENTIAL**
>
> This document contains detailed findings regarding the security posture of the target application. It may include information about vulnerabilities, architectural gaps, and potential exploitation vectors.
>
> **Access to this report is restricted to authorized stakeholders only.** Unauthorized distribution, copying, or public disclosure of this material is strictly prohibited and may compromise the security of the application.

---

## Summary

A brief overview of the security posture based on the audit.

**Coverage Statistics**:
- Total Level 1 Items: 70
- Items Verified: [Count] (Must be 70)
- **Result Breakdown**:
    - 🔴 Critical: [Count]
    - 🟠 High: [Count]
    - 🟡 Medium: [Count]
    - 🟢 Low: [Count]
    - ✅ PASS: [Count]
    - ⚠️ NEEDS_REVIEW: [Count]
- **Compliance Score**: [Percentage]% (Calculated as PASS / (Total Items - N/A Items - NEEDS_REVIEW Items) * 100)
- **Completeness Check**: [Total Reported] / [Total from CSV] (Should be 100%)
- **Review Debt**: [NEEDS_REVIEW Count] items require manual verification

## Findings

Only include detailed blocks for items that have a security finding (Critical, High, Medium, Low). NEEDS_REVIEW items should be captured in the Verification Summary, not as detailed Findings.

### #[Internal_Num] - [req_id] - [section_name]

- **Chapter**: [chapter_name]
- **Section**: [section_name]
- **ASVS ID**: [req_id]
- **Internal Item #**: [1-70]
- **Requirement**: [req_description] (FULL TEXT FROM CSV - DO NOT TRUNCATE)
- **Severity**: 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low
- **Location**: `path/to/file.ext:line_number`
- **Evidence**: `path/to/file.ext:line_number` and/or config key/value demonstrating the failure (or what is missing)
- **Description**: Detailed explanation of the vulnerability and why it fails the ASVS check.
- **Remediation**:
  To fix this issue, [steps to take].

  \`\`\`[language]
  // Corrected code example
  \`\`\`

---
(Repeat for each finding)

## Verification Summary

List all items that PASSED, are N/A, or NEEDS_REVIEW in this table.

| ASVS ID (#) | Chapter<br>Section | Requirement | Status | Evidence |
| :--- | :--- | :--- | :--- | :--- |
| # [Internal_Num] [req_id] | [chapter_name]<br>[section_name] | [req_description] (FULL TEXT) | ✅ PASS | `path/to/file.ext:line_number` or config key/value |
| # [Internal_Num] [req_id] | [chapter_name]<br>[section_name] | [req_description] (FULL TEXT) | ⚪ N/A | N/A - feature not present (brief reason) |
| # [Internal_Num] [req_id] | [chapter_name]<br>[section_name] | [req_description] (FULL TEXT) | ⚠️ NEEDS_REVIEW | `NEEDS_REVIEW - <reason>` |
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| CSV file missing/corrupted | STOP audit, report error: "ASVS CSV not found at expected path" |
| Target codebase empty | STOP audit, report: "No source files found in target repository" |
| Target codebase inaccessible | STOP audit, report: "Cannot access target path: [path]" |
| Git commands fail | Set Git Commit to `unknown`, continue audit |
| Tool fails mid-audit | Log the failed item as `FAIL` with severity 🟢 Low and note: "Verification failed due to tooling error". |
| Token/context limit approaching | Complete current chapter, save partial report with `[PARTIAL]` prefix, note last completed item |
| File too large to read | Sample first 500 lines + last 100 lines, note in Evidence: "Large file - sampled" |

---

## Appendices

### A. Quick Reference

Common pass/fail patterns for rapid evaluation:

| Pattern | Status | Evidence Format |
|---------|--------|----------------|
| Framework handles by default, no overrides found | ✅ PASS | `framework:<name>:<feature>` |
| Security library present and configured | ✅ PASS | `library:<name>@<version>:<feature>` |
| Secure config value explicitly set | ✅ PASS | `<filepath>#<key>=<value>` |
| Feature not used in codebase | ⚪ N/A | `N/A - <feature> not present` |
| Control missing, no library/middleware found | ❌ FAIL | `missing:<what_was_searched>` |
| Insecure function/pattern in code | ❌ FAIL | `<filepath>:<line>` |
| Default disabled or bypassed | ❌ FAIL | `<filepath>:<line>` (show bypass) |
| Documentation-only gap | ❌ FAIL (Low) | `missing:<docs_searched>` |
| Cannot verify due to tooling/limits | ⚠️ NEEDS_REVIEW | `NEEDS_REVIEW - <reason>` |

### B. Known Framework Defaults

These frameworks provide security controls by default. Mark as **✅ PASS** if using the framework AND no explicit disable/bypass is found:

| Framework | Default Security Feature | Bypass to Search For |
|-----------|-------------------------|----------------------|
| **React** | XSS protection (auto-escaping JSX) | `dangerouslySetInnerHTML` |
| **Angular** | XSS protection (auto-sanitization) | `bypassSecurityTrust*`, `[innerHTML]` |
| **Vue** | XSS protection (auto-escaping) | `v-html` directive |
| **Django** | CSRF protection | `@csrf_exempt`, `CSRF_COOKIE_SECURE=False` |
| **Django** | SQL injection protection (ORM) | `raw()`, `extra()`, `RawSQL()` |
| **Django** | XSS protection (auto-escaping) | `|safe`,`mark_safe()`,`{% autoescape off %}` |
| **Rails** | CSRF protection | `skip_before_action :verify_authenticity_token` |
| **Rails** | SQL injection protection (ActiveRecord) | `find_by_sql`, string interpolation in `where()` |
| **Rails** | XSS protection (auto-escaping) | `raw()`, `html_safe`, `<%==` |
| **Spring Boot** | CSRF protection | `csrf().disable()` |
| **Spring Boot** | SQL injection (JPA/Hibernate) | Native queries with concatenation |
| **ASP.NET Core** | XSS protection (Razor auto-encoding) | `@Html.Raw()` |
| **ASP.NET Core** | CSRF protection (antiforgery) | `[IgnoreAntiforgeryToken]` |
| **Express + Helmet** | Security headers | Check `helmet()` middleware is applied |
| **Next.js** | XSS protection (React-based) | `dangerouslySetInnerHTML` |
| **Laravel** | CSRF protection | `@csrf` missing, `VerifyCsrfToken` middleware excluded |
| **Laravel** | SQL injection (Eloquent ORM) | `DB::raw()`, `whereRaw()` with user input |

### C. Contextual Triggers

This skill should automatically activate or be suggested when:

- The user asks for a "security audit" or "security review".
- The user references "OWASP" or "ASVS".
- The user asks to "check for vulnerabilities".
- Critical security-sensitive code (auth logic, crypto, input handling) is selected or shared.

---

## Examples

### Example Request

**User:** "Run a security audit on this project."

### Example Report Header & Statistics

```markdown
# Acme Corp · OWASP ASVS 5.0 Level 1 · Security Audit Report

---

**Initial Draft author**: AI Agent (Claude Sonnet 4 via GitHub Copilot)  
**Reviewed & Finalized by**: [Security Auditor Name] [auditor.email@acme.com]  
**Report Date**: 2026-01-25  
**Skill Version**: 1.4.0  
**ASVS Version**: 5.0.0  

## Application Details

**App Version**: 2.3.1  
**Git Commit**: a1b2c3d  

## Tech Stack

| Attribute | Value |
|-----------|-------|
| **Language** | TypeScript |
| **Framework** | Next.js 14 |
| **Database** | PostgreSQL (Prisma) |
| **Key Libraries** | NextAuth.js, zod, bcrypt, helmet |

---

## Summary

**Coverage Statistics**:
- Total Level 1 Items: 70
- Items Verified: 70
- **Result Breakdown**:
    - 🔴 Critical: 1
    - 🟠 High: 3
    - 🟡 Medium: 5
    - 🟢 Low: 3
    - ✅ PASS: 51
    - ⚪ N/A: 5
    - ⚠️ NEEDS_REVIEW: 2
- **Compliance Score**: 82.3% (51 PASS / 62 evaluable items)
- **Completeness Check**: 70 / 70 (100%)
- **Review Debt**: 2 items require manual verification
```

### Example: ❌ FAIL Finding

```markdown
### #12 - V2.1.2 - Password Security

- **Chapter**: V2 Authentication
- **Section**: V2.1 Password Security
- **ASVS ID**: V2.1.2
- **Internal Item #**: 12
- **Requirement**: Verify that passwords of at least 64 characters are permitted, and that passwords of more than 128 characters are denied.
- **Severity**: 🟡 Medium
- **Location**: `src/lib/auth/validation.ts:23`
- **Evidence**: `src/lib/auth/validation.ts:23` - `maxLength: 50` restricts passwords to 50 characters
- **Description**: The password validation schema enforces a maximum length of 50 characters, which is below the ASVS requirement of permitting at least 64 characters. This unnecessarily limits password strength and may prevent users from using passphrases.
- **Remediation**:
  Update the password validation to allow 64-128 characters:

  ```typescript
  // src/lib/auth/validation.ts
  export const passwordSchema = z.string()
    .min(12, "Password must be at least 12 characters")
    .max(128, "Password must not exceed 128 characters"); // Changed from 50
  ```

```

### Example: ✅ PASS Entry (in Verification Summary)

```markdown
| ASVS ID (#) | Chapter<br>Section | Requirement | Status | Evidence |
| :--- | :--- | :--- | :--- | :--- |
| #15 V2.2.1 | V2 Authentication<br>V2.2 General Authenticator Security | Verify that anti-automation controls are effective at mitigating breached credential testing, brute force, and account lockout attacks. | ✅ PASS | `library:next-auth@5.0:built-in-rate-limiting` + `src/app/api/auth/[...nextauth]/route.ts:12` custom lockout after 5 attempts |
```

### Example: ⚪ N/A Entry (in Verification Summary)

```markdown
| ASVS ID (#) | Chapter<br>Section | Requirement | Status | Evidence |
| :--- | :--- | :--- | :--- | :--- |
| #45 V12.1.1 | V12 Files and Resources<br>V12.1 File Upload | Verify that the application will not accept large files that could fill up storage or cause a denial of service. | ⚪ N/A | N/A - no file upload endpoints found (searched: `upload`, `multer`, `formidable`, `multipart` in src/) |
```
