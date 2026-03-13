---
name: test-plan
description: Write a comprehensive test plan for a feature or system. Use when users want to create a test plan, QA strategy, or testing checklist for a feature, bug fix, API, UI component, or release. Trigger when a feature, spec, PRD, or ticket is shared and the user asks what or how to test.
category: "Documentation"
---

# Test Plan Skill

Generate a thorough, well-structured test plan for any software feature, system, or change.

## Goal

Produce a test plan that a QA engineer or developer can immediately act on — one that covers the right scope, flags risk areas, and is specific enough to actually guide test execution.

## Process

### Step 1: Gather Context

Before writing anything, collect what you need. If the user has shared a spec, ticket, or description, extract from it. Otherwise ask for what's missing. Key questions:

- **What is the feature?** What does it do, what problem does it solve?
- **Who are the users / actors?** Who triggers this behavior?
- **What's the tech stack / surface?** (API, UI, mobile, background job, etc.)
- **What already exists?** Is this new or modifying existing behavior?
- **What's the risk level?** (Low-stakes internal tool vs. payment flow)
- **Any known constraints or dependencies?** (Auth, third-party services, feature flags)

If the user wants to get started quickly without answering questions, proceed with reasonable assumptions and call them out explicitly.

---

### Step 2: Write the Test Plan

Structure the output as follows. Adapt section depth to the feature's complexity — a simple CRUD endpoint needs less than a multi-actor checkout flow.

---

## Test Plan Template

```
# Test Plan: [Feature Name]

## Overview
One paragraph: what this feature does, why it matters, and what this test plan covers.

## Scope

### In Scope
- List of behaviors, flows, and components that will be tested

### Out of Scope
- Explicitly excluded areas (and why, if non-obvious)

## Test Approach

Brief narrative on the testing strategy: which test types apply, any tooling or environment notes, and how coverage is prioritized.

## Test Types

List which of the following apply and why:
- **Unit tests** — isolated logic, functions, pure computation
- **Integration tests** — interactions between components or services
- **End-to-end (E2E) tests** — full user flows through the system
- **API tests** — contract, request/response validation
- **UI/UX tests** — visual correctness, accessibility, responsiveness
- **Performance tests** — load, latency, throughput (flag if relevant)
- **Security tests** — auth, authorization, injection, data exposure
- **Regression tests** — existing behavior that must not break

## Test Scenarios

Organize by functional area or user flow. For each scenario:

| # | Scenario | Steps | Expected Result | Priority |
|---|----------|-------|-----------------|----------|
| 1 | Happy path: [description] | 1. ... 2. ... | ... | High |
| 2 | ... | | | |

Priority: High (must pass for launch) / Medium (should pass) / Low (nice to have)

## Edge Cases & Negative Tests

- Input validation: empty, null, very long, special characters, wrong types
- Boundary values: limits, maximums, minimums
- Concurrency: simultaneous requests, race conditions
- State transitions: invalid state changes, double-submits
- Failure modes: network errors, timeouts, third-party failures
- Permission edge cases: wrong role, expired session, cross-tenant access

## Acceptance Criteria

What must be true for this feature to be considered "done" from a testing perspective? Map back to the feature's stated requirements.

## Risks & Open Questions

- Known unknowns that need clarification before or during testing
- High-risk areas deserving extra attention
- Dependencies on external systems or data

## Test Data Requirements

What data is needed? (e.g., test accounts, specific DB states, mocked services)

## Environment Notes

Any specific setup, feature flags, or config needed to run these tests.
```

---

### Step 3: Calibrate to Context

After generating the plan, briefly flag:

- **Top 3 highest-risk scenarios** (if not obvious from the plan)
- **Any gaps** you couldn't fill without more info
- **Suggested priority order** if the team needs to triage

---

## Quality Bar

A good test plan from this skill should:
- Be specific enough that someone unfamiliar with the feature could execute it
- Cover both the happy path and realistic failure modes
- Make explicit what is *not* being tested (and why)
- Be proportional — don't pad a simple feature with 40 low-value test cases
- Use plain language; avoid vague phrases like "verify it works correctly"

## Format Notes

- Default output: Markdown (renders well in Notion, Linear, GitHub, Confluence)
- If the user wants a Word doc or other format, use the appropriate skill
- Tables are preferred for test scenarios; prose for narrative sections
- Keep scenario descriptions action-oriented: "User submits form with missing required field" not "Test form validation"