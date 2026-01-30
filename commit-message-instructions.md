# Commit Message Guidelines

## Core Principles

- **Clarity**: Messages must clearly explain *what* changed and *why*.
- **Security**: 🚫 NEVER include secrets, API keys, credentials, or PII in commit messages.
- **Consistency**: Follow the Conventional Commits specification strictly.

## Format Structure

```text
<type>(<scope>): <subject>

<body>

<footer/metadata>

<file-changes>
```

## 1. Header Line

The header must be **less than 72 characters**.

### Types

Select the most specific type:

- `feat`: 🚀 New feature (correlates with MINOR in semantic versioning)
- `fix`: 🐛 Bug fix (correlates with PATCH in semantic versioning)
- `docs`: 📚 Documentation changes
- `style`: 💎 Code style/formatting (no logic change, whitespace, semi-colons)
- `refactor`: 📦 Code refactoring (no functional change, no api change)
- `perf`: 🚀 Performance improvements
- `test`: 🚨 Adding or updating tests
- `chore`: 🛠️ Maintenance tasks, dependencies, build scripts
- `ci`: ⚙️ CI/CD configuration changes
- `sec`: 🔒 Security fixes or improvements

### Scope (Optional)

Use the affected module/component (folder) name (lowercase, kebab-case):

- Core: `api`, `auth`, `data`, `utils`
- Infra: `ci`, `config`, `deploy`, `scripts`
- UI: `components`, `styles`, `views`, `assets`

### Subject

- Use **imperative mood** ("Add feature" NOT "Added feature").
- Capitalize the first letter.
- Do not end with a period.
- Be concise but descriptive.

## 2. Body

- **Mandatory** for all `feat`, `fix`, and complex `refactor` changes.
- Separate from subject with a blank line.
- Wrap lines at 72 characters.
- Explain the **motivation** for the change and contrast with previous behavior.
- Use bullet points (`-`) for lists.

## 3. Footer / Metadata

- Reference issue tracker IDs explicitly (e.g., Jira, GitHub Issues).
- Format: `Ref: #123` or `Fixes: ISSUE-123`.
- Mention breaking changes if any: `BREAKING CHANGE: <description>\`.

## 4. File Changes Summary

Include a high-level summary of changes per file using gitmoji to aid visual scanning.

| Gitmoji | Meaning | Context |
|---------|---------|---------|
| ✨ | New feature | Logic, Use cases |
| 🐛 | Bug fix | Logic correction |
| ♻️ | Refactor | Cleanup, Simplification |
| 🎨 | Style | CSS, Formatting |
| 📝 | Documentation | README, Comments |
| 🔧 | Configuration | Config files |
| ✅ | Tests | Unit tests, Integration tests |
| 🗑️ | Removal | Deprecated code deletion |
| 🚚 | Move/rename | File organization |
| 🔒 | Security | Auth, Sanitization, Secrets |

### Example

```text
feat(auth): Add login rate limiting

Implement rate limiting on the login endpoint to prevent brute
force attacks. Users are now locked out after 5 failed attempts.

Ref: ISSUE-456

---
✨ src/auth/login-service.ts - Add rate limiter logic
🔧 config/security.yaml - Configure max attempts
✅ tests/auth/login.spec.ts - Add tests for lockout
```
