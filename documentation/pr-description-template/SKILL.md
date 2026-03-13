---
name: pr-description-template
description: Generate a clear, structured PR description from a diff, commit list, branch name, or change summary. Use when users want to write or improve a pull request description, fill a PR template, or document code changes for review. Includes what changed, why, testing steps, risks, and follow-ups.
category: "Documentation"
---

# PR Description Generator

Generate clear, complete, reviewer-friendly PR descriptions from raw inputs like diffs, commit logs, branch names, or free-form summaries.

---

## Input Formats (accept any combination)

- **Git diff** (`git diff main...feature-branch` or paste of diff output)
- **Commit list** (`git log --oneline` output, or a list of commit messages)
- **Branch name** (e.g. `fix/auth-token-expiry`, `feat/add-dark-mode`)
- **Free-form description** ("I added retry logic to the payment service and fixed a bug in the logger")
- **Existing partial PR** (user has a rough draft they want improved)

If input is minimal or ambiguous, make reasonable inferences and note assumptions. Do not ask clarifying questions unless something critical is truly unknowable (e.g. "what is the intent of this change?").

---

## Output Structure

Always produce a PR description with these sections. Use Markdown. Omit a section only if it genuinely does not apply (e.g. no migration steps for a pure refactor).

### Title
A single line: `[type]: <short imperative summary>` (50 chars max)
- Types: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `perf`, `security`
- Example: `fix: prevent token refresh loop on 401 response`

---

### Summary
2–4 sentences covering:
- **What** changed (the "what" at a high level, not a list of files)
- **Why** it was needed (the motivation, bug, or user need)
- **How** it was solved (the approach taken)

Keep it scannable. Avoid jargon unless the codebase clearly uses it.

---

### Changes
A bullet list of the key changes. Group by area if more than ~5 items. Be specific but concise.

```
- Added retry logic with exponential backoff to `PaymentService.charge()`
- Extracted `TokenRefresher` class to isolate refresh concerns
- Updated `AuthMiddleware` to pass refresh errors downstream instead of swallowing them
```

Do NOT just list filenames. Describe what actually changed and why.

---

### How to Test
Step-by-step instructions a reviewer can follow to verify the change works. Include:
- Setup steps (env vars, seed data, feature flags)
- Manual test scenarios with expected outcomes
- Any automated tests added or modified

```
1. Run `npm test -- --watch auth` to see the new token refresh tests pass
2. Start the dev server: `npm run dev`
3. Log in, let your session expire (or set TOKEN_TTL=5 in .env)
4. Confirm you're redirected to /login rather than seeing a blank screen
```

---

### Risks & Considerations
Flag anything reviewers should think carefully about:
- Breaking changes or API surface changes
- Performance implications
- Security considerations
- Rollback complexity
- Areas of uncertainty or known shortcuts taken

If there are no real risks, write: `No significant risks identified.`

---

### Follow-ups (optional)
List any known TODOs, deferred work, or tech debt created:
```
- [ ] Add integration test for concurrent refresh race condition (#412)
- [ ] Migrate legacy `refreshToken()` callers to new interface
```

Omit this section if there are no follow-ups.

---

## Tone & Style

- **Audience**: Engineers who will review the code, not the original author
- **Voice**: Direct, precise, present tense ("adds", "fixes", "removes")
- **Length**: As long as needed — don't pad, don't truncate important context
- **Avoid**: Vague phrases like "various improvements", "misc fixes", "updated stuff"

---

## Working with Diffs

When given a raw diff:
1. Scan for the **shape** of the change (new files, deleted files, modified functions, config changes)
2. Identify the **dominant intent** (bug fix? new feature? refactor? dependency update?)
3. Look for **test files** — they reveal expected behavior and confirm intent
4. Note any **migration files**, **schema changes**, or **config updates** — flag these explicitly as risks

For large diffs (>500 lines), focus on:
- Entry points and public interfaces
- Files changed the most
- Any `BREAKING CHANGE` in commit messages

---

## Example Output

**Input**: `fix/null-check-user-profile` branch + diff showing a null guard added to `UserProfile.render()`

**Output**:

---

**fix: prevent crash when user profile photo is null**

### Summary
User profiles with no photo set were causing a runtime crash in the profile renderer. This fix adds a null guard before accessing `photo.url`, falling back to a default avatar. The issue was introduced in #388 when we migrated to the new profile API that made `photo` optional.

### Changes
- Added null check for `user.photo` in `UserProfile.render()`
- Added fallback to `DEFAULT_AVATAR_URL` constant when photo is absent
- Added unit test covering null, undefined, and empty-string photo values

### How to Test
1. Create a test user with no profile photo (or use `seed:no-photo` fixture)
2. Navigate to `/profile/:userId`
3. Confirm the default avatar renders without a JS error in the console

### Risks & Considerations
- Minimal — this is a pure defensive fix with no API changes
- The `DEFAULT_AVATAR_URL` constant is hardcoded; may want to make it configurable later

### Follow-ups
- [ ] Audit other places `user.photo` is accessed for similar null risks

---

## Notes

- If the user provides a team-specific PR template, fill it in rather than using this structure
- If the PR is a work-in-progress (WIP/draft), note that in the title: `[WIP] feat: ...`
- For dependency bumps, always check if there are breaking changes and mention them