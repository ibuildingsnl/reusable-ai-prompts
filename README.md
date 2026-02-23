# AI Agent Reusable Prompts

This repository manages custom agent skills, prompt engineering templates, and development guidelines for AI coding assistants like GitHub Copilot.

## Main Purpose

- **Skills**: Modular capabilities that extend the agent's functionality.
- **Agents**: Specialized personas and prompts for specific tasks.
- **Guidelines**: Standards for consistent development practices.

## Agents Prompt

Specialized agent definitions are stored in the `agents/` directory.

### [Prompt Engineer](agents/prompt-engineer.agent.md)

**Name**: `Prompt Engineer`
**Description**: Expert guidance on crafting effective, safe, and robust AI prompts using best practices like CoT, few-shot, and personas.

## Available Skills

### [ASVS Audit](skills/asvs-audit/SKILL.md)

**Name**: `asvs-audit`
**Description**: Performs a comprehensive security audit of the provided codebase against the OWASP Application Security Verification Standard (ASVS) 5.0 Level 1. Use this when asked for a asvs or security audit.

### [GitLab Merge Request Review](skills/gitlab-mr-review/SKILL.md)

**Name**: `gitlab-mr-review`
**Description**: Review a GitLab Merge Request and provide findings, and post structured review comments with issue explanation plus pseudo code fixes. Use this skill when asked to review a GitLab merge request.

### [Implementation Plan](skills/implementation-plan/SKILL.md)

**Name**: `implementation-plan`
**Description**: Create a technical implementation plan with time estimation. Use this skill when asked for an implementation plan, technical breakdown, or task estimation.

## Git Commit Message

This repository includes a commit message template for use with AI's and to ensure clarity and consistency. See [Commit Message Guidelines](commit-message-instructions.md) for full details.
