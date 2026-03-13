---
name: git-workflow-architect
description: Design and document a complete Git workflow for a team, including branching strategy, commit conventions, pull request process, merge rules, CI requirements, and release strategy. Use when creating Git workflow docs, CONTRIBUTING guidelines, or defining version control processes for a project.
category: "Documentation"
---

# Git Workflow Documentation Skill

## Overview
This skill produces a complete, well-structured Git workflow document for a software team. The output is a professional Markdown (`.md`) or Word (`.docx`) document covering branching strategy, commit message format, pull request process, and merge rules — tailored to the team's actual setup.

## Trigger Conditions
Use this skill when the user asks to:
- Document or write up their Git workflow
- Create a Git branching guide or strategy doc
- Define or formalize commit message conventions
- Document their PR (pull request) or code review process
- Write merge rules or branch protection policies
- Onboard new engineers to their version control process
- Create a CONTRIBUTING.md or equivalent developer guide

**Example triggers:**
- "Document our Git workflow"
- "Write a branching strategy doc for our team"
- "Create a guide for commit messages and PRs"
- "Help me write a CONTRIBUTING.md"
- "We use Gitflow — document it for us"

---

## Step-by-Step Instructions

### Step 1: Gather Team Context (if not provided)

Before writing, Claude should identify the following. If the user hasn't provided details, ask in a single consolidated question using the `ask_user_input` tool:

**Key questions to resolve:**
1. **Branching model** — Gitflow, GitHub Flow, trunk-based, or custom?
2. **Main branches** — What are the long-lived branches? (e.g., `main`, `develop`, `staging`)
3. **Feature branch naming** — Any conventions? (e.g., `feature/`, `fix/`, `chore/`)
4. **Commit message format** — Conventional Commits, custom format, or no standard yet?
5. **PR process** — Required reviewers, CI checks, draft PRs?
6. **Merge strategy** — Squash, merge commit, rebase?
7. **Deployment model** — Does merging to `main` trigger a deploy?

If the user says "just use sensible defaults", use **GitHub Flow** as the default.

---

### Step 2: Select Output Format

- Default to **Markdown** (`.md`) — ideal for storing in the repo
- If the user asks for a Word document, read `/mnt/skills/public/docx/SKILL.md` and produce a `.docx`

---

### Step 3: Write the Document

Produce a complete document with these sections:

**1. Overview** — Workflow philosophy summary

**2. Branch Structure** — Table of branches and their purpose

**3. Branch Naming Conventions**
- Format: `<type>/<short-description>` in kebab-case
- Types: `feature`, `fix`, `hotfix`, `chore`, `docs`, `refactor`, `release`
- Examples: `feature/user-auth`, `fix/login-redirect-bug`

**4. Commit Message Format**
- Default to Conventional Commits: `<type>(<scope>): <description>`
- Include a full example with body and footer
- Rules: 72 char subject limit, imperative mood, issue references

**5. Pull Request Process**
- Creating a PR, PR title/description format, draft PRs
- Review requirements (approvals, CI checks)
- Review etiquette (constructive feedback, "nit:" prefix, resolving comments)

**6. Merge Rules**
- Merge strategy (squash / merge commit / rebase)
- Who can merge, branch deletion policy, protected branches

**7. Release Process** *(if applicable)*
- Semantic versioning tags, changelog generation

**8. Hotfix Process**
- Branch off `main`, PR directly into `main`, tag immediately

**9. Tips & Common Mistakes** *(optional)*

---

### Step 4: Format and Deliver

- Save as `git-workflow.md` or `CONTRIBUTING.md`
- Move to `/mnt/user-data/outputs/` and share with `present_files`

---

## Quality Checklist

- [ ] All 6+ core sections present
- [ ] Branch naming includes concrete examples
- [ ] Commit message section has a full worked example
- [ ] PR process covers review requirements and CI
- [ ] Merge strategy is unambiguous
- [ ] Written in second person or imperative voice
- [ ] No unfilled placeholder text
- [ ] File delivered via `present_files`

---

## Default Values

| Setting | Default |
|---|---|
| Branching model | GitHub Flow |
| Main branch | `main` |
| Feature branches | `feature/<name>` off `main` |
| Commit format | Conventional Commits |
| Merge strategy | Squash and merge |
| Required reviewers | 1 approval |
| Branch deletion | Auto-delete after merge |
| Protected branches | `main` |

---

## Example Output File Names

- `CONTRIBUTING.md` — team onboarding doc in repo root
- `docs/git-workflow.md` — internal team reference
- `git-workflow.docx` — for sharing outside the codebase