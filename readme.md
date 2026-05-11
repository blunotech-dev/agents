# blunotech-dev/agents

> Curated agent skills for AI coding assistants. Hand-vetted. No bloat.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![npxskills](https://img.shields.io/badge/browse-npxskills.xyz-black)](https://npxskills.xyz)
[![Skills](https://img.shields.io/badge/skills-12%20categories-blue)](#skills)
[![Agents](https://img.shields.io/badge/agents-Claude%20%7C%20Cursor%20%7C%20Copilot%20%7C%2014%2B-purple)](#supported-agents)

---

## Why this exists

Existing skill directories have more than 500,000 entries. We still couldn't find one we actually wanted to use.

This repo is the opposite: small, deliberate, and hand-vetted. Every skill here has been tested against real workflows. Nothing gets added just to grow the number.

**Browse the full catalog → [npxskills.xyz](https://npxskills.xyz)**

---

## Quick install

```bash
# Install a specific skill into your project
npx skills add blunotech-dev/agents --skill auth-state-sync

# Install an entire category
npx skills add blunotech-dev/agents/frontend

# Install all skills from this repo
npx skills add blunotech-dev/agents
```

Skills are installed to your agent's configured directory (e.g., `.claude/skills/` for Claude Code, `.github/skills/` for Copilot) and activate automatically when relevant.

---

## Skills

| Category | Skills |
|---|---|
| [`ai integration`](./ai%20integration/) | LLM API patterns, prompt chaining, model selection, streaming |
| [`backend`](./backend/) | API design, server architecture, database queries, auth flows |
| [`debugging`](./debugging/) | Systematic triage, root cause analysis, error recovery workflows |
| [`design`](./design/) | UI/UX guidance, component patterns, design systems |
| [`documentation`](./documentation/) | READMEs, changelogs, API docs, inline comments |
| [`frontend`](./frontend/) | Component architecture, accessibility, responsive design |
| [`fullstack`](./fullstack/) | End-to-end feature delivery, data flow, deployment patterns |
| [`rapid mvp building`](./rapid%20mvp%20building/) | Fast prototyping workflows with minimal overhead |
| [`react`](./react/) | Hooks, state management, rendering patterns, performance |
| [`refactor`](./refactor/) | Safe, incremental refactoring — without breaking things |
| [`security`](./security/) | Input validation, auth hardening, common vulnerability patterns |
| [`testing`](./testing/) | Unit, integration, and E2E testing workflows |

---

## How skills work

Each skill is a folder with a `SKILL.md` file — plain markdown with YAML frontmatter:

```
frontend/
└── SKILL.md      ← name, description, instructions for the agent
```

Agents load skills **on demand**. Only the skills relevant to your current task activate, keeping context usage low. The full skill content loads into context only when the agent decides it's needed.

```yaml
---
name: frontend
description: >
  Component architecture, design systems, state management, and WCAG 2.1 AA
  accessibility patterns. Use when building or reviewing any frontend UI.
---

# Frontend Skill

...instructions...
```

Skills are plain markdown. No proprietary format. No lock-in. Portable across any LLM.

---

## Supported agents

Works with any agent that supports the [Agent Skills specification](https://github.com/skillmatic-ai/awesome-agent-skills):

| Agent | Install path |
|---|---|
| [Claude Code](https://claude.ai/code) | `.claude/skills/` |
| [Cursor](https://cursor.sh) | `.cursorrules/skills/` |
| [GitHub Copilot](https://copilot.github.com) | `.github/skills/` |
| [Cline](https://github.com/cline/cline) | `.cline/skills/` |
| [Windsurf](https://codeium.com/windsurf) | `.windsurf/skills/` |
| [OpenCode](https://opencode.ai) | `.opencode/skills/` |
| Codex, Amp, Roo Code, Goose + 6 more | agent-specific paths |

The `npx skills` CLI auto-detects which agents you have installed and installs to the right path.

---

## What makes a skill good

We keep this catalog small on purpose. A skill earns a spot here by being:

- **Specific** — triggers on clear intent, not vague patterns
- **Concise** — under 500 lines; references go in separate files
- **Tested** — validated against real workflows, not synthetic prompts
- **Portable** — works across agents without agent-specific hacks

---

## Built by

[Sweekar Koirala](https://www.linkedin.com/in/sweekarkoirala/) and [Aditya Thapa Magar](https://www.linkedin.com/in/aditya-thapa-magar-6a2912288/).

Questions, feedback, or ideas → open an [issue](https://github.com/blunotech-dev/agents/issues).

---

## License

[MIT](./LICENSE) — free to use, modify, and distribute.

Skills in this repo are a mix of original work and MIT/Apache-2.0 licensed material. Attribution is included in individual skill files where applicable.
