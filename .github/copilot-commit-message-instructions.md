# Commit Message Guidelines

## Format

Use the conventional commit format:
```
<type>(<scope>): <subject>

<body>

<ticket-reference>

<file-changes>
```

## Types

Use one of the following types:
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation changes
- `style` - Code style/formatting (no logic change)
- `refactor` - Code refactoring
- `perf` - Performance improvements
- `test` - Adding or updating tests
- `chore` - Maintenance tasks, dependencies
- `ci` - CI/CD configuration changes

## Scope (optional)

Use the affected skill or component:
- **Global**: `repo`, `docs`, `config`
- **Skills**: `asvs-audit`, `implementation-plan`

## Subject Line

- Use imperative mood ("Add feature" not "Added feature")
- Limit to 72 characters
- Capitalize the first letter
- Do not end with a period

## Body

- Separate from subject with a blank line
- Explain **what** changed and **why**, not how

## Ticket Reference

If the branch name follows the pattern `type/[ticket-number]-[title]` (e.g., `feature/AIP-123-new-login`), include the ticket number:

- Format: `Refs: #<ticket-number>` or `Closes: #<ticket-number>`
- Place it below the body, separated by a blank line

## File Changes Section

Include a summary of changes per file using gitmoji:

| Gitmoji | Meaning |
|---------|---------|
| ✨ | New feature |
| 🐛 | Bug fix |
| ♻️ | Refactor |
| 🎨 | Style/format |
| 📝 | Documentation |
| 🔧 | Configuration |
| ✅ | Tests |
| 🗑️ | Removal |
| 🚚 | Move/rename |

### Example

``
feat(asvs-audit): Update compliance checklist

Update the audit criteria to include new authentication checks.
This improves the accuracy of the security audit results.

Relates to #42

---
✨ skills/asvs-audit/SKILL.md - Update audit instructions
📝 skills/asvs-audit/assets/OWASP_Application_Security_Verification_Standard_5.0.0_L1_en.csv - Update checklist data
```
