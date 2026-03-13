---
name: readme-generator
description: Generate a structured, professional README.md from a project's files, such as file structure, manifests (e.g., package.json), and entry points. Use when users want to create or improve a README, document a project, or explain a codebase. Includes project purpose, tech stack, setup, usage, API/CLI reference, environment variables, folder structure, and contribution guidelines.
category: "Documentation"

---

# README Generator

Generate a clear, developer-friendly README.md by analyzing a project's actual files —
not generic boilerplate. The output should reflect what the project *actually does*.

---

## Step 1 — Gather Project Context

Before writing anything, collect as much signal as possible. Read these files if present:

| Priority | Files to read |
|----------|---------------|
| High | `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `composer.json` |
| High | `index.js`, `main.py`, `app.py`, `src/index.*`, `cmd/main.go`, `src/main.*` |
| Medium | `.env.example`, `.env.sample`, `config.example.*` |
| Medium | `Makefile`, `Dockerfile`, `docker-compose.yml` |
| Medium | `CONTRIBUTING.md`, `LICENSE`, `CHANGELOG.md` (if exists, link don't duplicate) |
| Low | `src/` or `lib/` folder structure (top 2 levels only) |

Use `bash_tool` to list the project structure:
```bash
find . -maxdepth 3 -not -path '*/node_modules/*' -not -path '*/.git/*' \
       -not -path '*/__pycache__/*' -not -path '*/dist/*' -not -path '*/build/*' \
       | sort | head -80
```

Also read the entry point file (first 60–80 lines) to understand what the app does.

---

## Step 2 — Infer Project Intent

From what you've gathered, determine:

- **What does this project do?** (in 1–2 plain sentences)
- **Who is it for?** (CLI tool, library, web app, API service, script, etc.)
- **What's the tech stack?** (language, framework, key deps)
- **How do you run it?** (scripts in package.json, Makefile targets, CLI commands)
- **Are there environment variables?** (from `.env.example` or config files)

If there's an existing partial README, read it and *improve* it — don't replace content
that's already accurate or detailed.

---

## Step 3 — Write the README

Use this section structure. **Skip sections that don't apply.** Don't add empty headings.

```markdown
# Project Name

> One-line tagline describing what it does.

Brief paragraph (2–4 sentences) expanding on purpose and key use case.

## Features

- Bullet list of notable capabilities (skip if obvious or trivial)

## Requirements

- Runtime versions, OS requirements, required services (DB, Redis, etc.)

## Installation

​```bash
# Commands to install / clone / set up
​```

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `VAR_NAME` | What it controls | `value` |

(Only include if there are env vars or config files)

## Usage

​```bash
# Primary usage command(s)
​```

Include 2–3 real examples if the CLI/API has notable options.

## Project Structure

​```
src/
├── index.js       # Entry point
├── routes/        # Express route handlers
└── utils/         # Shared utilities
​```

(Only include if structure is non-obvious; keep to ~10 lines max)

## API Reference

Brief table or list of public exports / endpoints (only for libraries/APIs)

## Development

​```bash
# How to run in dev mode, run tests, lint
​```

## Contributing

Brief guide or link to CONTRIBUTING.md

## License

[MIT](LICENSE) — or whatever the LICENSE file says
```

---

## Style Rules

- **Concrete over generic.** Use actual project names, real command names, real env var names.
- **Code blocks for all commands.** Never describe a command in prose if you can show it.
- **No filler.** Don't write "This project is designed to..." or "Feel free to...".
- **Infer don't hallucinate.** If you can't find the install command, say `# TODO: add install command` rather than guessing.
- **Short is fine.** A 30-line README that's accurate beats a 200-line one with padding.
- **Badges** are optional. Only add them if the project has CI/npm/PyPI that you can confirm.

---

## Step 4 — Output

Write the final README as a **`README.md` file** in the project root (or `/mnt/user-data/outputs/README.md` if working in the container). Then call `present_files` so the user can download it.

If you're unsure about any section (e.g., license, test command), add a `<!-- TODO: verify -->` comment so the developer knows to review it.

---

## Edge Cases

| Situation | What to do |
|-----------|------------|
| No manifest file found | Infer stack from file extensions; note uncertainty |
| Monorepo | Document the root, then list packages with one-line descriptions |
| Existing README present | Read it first; improve/expand rather than replace |
| Private/internal tool | Skip badges, keep contribution section minimal |
| Library/package | Emphasize API reference and import examples |
| CLI tool | Lead with usage examples; show `--help` output if inferable |