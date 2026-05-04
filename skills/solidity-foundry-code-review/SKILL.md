---
name: solidity-foundry-code-review
description: "Review Solidity contracts in Foundry projects for engineering quality before merge or after feature work. Use when users want file-grounded findings about correctness, edge cases, maintainability, interface design, call flows, deployment assumptions, dependency integration, or test gaps without requesting a full security audit or optimization pass."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Review Solidity and Foundry Code Quality

## Core Workflow

### Understand the Request Before Reviewing

Use this skill when the user asks for a code review, merge review, engineering review, PR review, or post-development quality check for Solidity / Foundry code.

If the request is explicitly a security audit, use the security review skill. If it asks to add tests, use the Foundry test skill. If it asks for gas or maintainability optimization after development, use the optimization skill. A code review may identify those areas, but should not expand into a full audit, test-writing pass, or optimization pass unless the user asks.

### Always Read the Project First

Before reporting findings:

1. Read `README.md`, `foundry.toml`, and project docs when they exist.
2. Search `src/`, `test/`, `script/`, interfaces, libraries, mocks, fixtures, and deployment helpers.
3. Read the files in scope and neighboring files needed to understand behavior across contract boundaries.
4. Read tests to infer expected behavior, but do not assume tests are complete or correct.
5. Read `remappings.txt`, `lib/`, `node_modules/`, and package config when imports, inheritance, or dependency behavior matters.

If a file cannot be read, surface the exact path and explain how that limits review confidence. Do not fall back to a generic checklist as if the project was reviewed.

### Review Posture

Default to a code-review stance:

- Lead with actionable findings, ordered by severity.
- Prioritize correctness, behavioral regressions, unclear assumptions, missing validation, state-flow mistakes, integration mistakes, and test gaps.
- Ground every finding in specific files and call paths.
- Avoid style nits unless they create real maintenance risk.
- Do not report speculative issues as strong findings. Lower confidence or list them as residual risk.

## Review Method

### Step 1: Reconstruct Intent and Scope

Build a compact model of:

- user-facing and privileged entrypoints
- expected state transitions and call sequences
- assets, balances, roles, configuration, and deployment assumptions
- external integrations and dependency-provided behavior
- tests that define or imply intended behavior

When reviewing a diff, compare the changed behavior against the existing system. When reviewing the whole project, identify the contracts and flows that matter most.

### Step 2: Check Correctness and Edge Cases

Look for project-specific issues such as:

- invalid or missing preconditions
- state updates in the wrong order
- inconsistent accounting or stale cached values
- boundary errors around zero values, max values, empty arrays, duplicate inputs, rounding, decimals, time, or block assumptions
- permission checks that differ across equivalent paths
- events that are missing, misleading, or emitted at the wrong time
- revert behavior that makes integrations brittle or ambiguous
- deployment or initialization paths that can leave the system partially configured

This is not a vulnerability checklist. Tie each issue to the project's intended behavior.

### Step 3: Check Interfaces and Integration

Review how contracts expose and consume behavior:

- public and external APIs: naming, return values, revert surface, caller expectations, and backwards compatibility
- internal boundaries: helper responsibilities, duplicated logic, and unclear abstractions
- dependency integration: required overrides, hooks, modifiers, constructors, initializers, storage, and version-specific behavior
- scripts and deployment helpers: config consistency, initialization order, environment assumptions, and repeatability
- tests and mocks: whether they exercise realistic user and operator paths

Read the installed dependency source before making claims about OpenZeppelin, Solady, Chainlink, Uniswap, proxy libraries, token standards, or other imported behavior.

### Step 4: Separate Review Types

Classify findings by the right next action:

- **Code review finding** - correctness, interface, maintainability, or integration issue that should be fixed before merge.
- **Security follow-up** - possible exploit or trust-boundary issue that needs the security review skill or deeper adversarial analysis.
- **Test follow-up** - missing or weak test coverage that should be handled by the Foundry test skill.
- **Optimization follow-up** - gas, structure, or maintainability improvement better handled by the optimization skill.

Do not blur these categories. A code review can route work without pretending to complete every specialized review.

### Step 5: Recommend Minimal Fixes and Tests

For each actionable finding:

- Describe the smallest behavior-preserving fix.
- Prefer project patterns and installed dependencies over new custom abstractions.
- Note any storage-layout, initializer, ABI, or integration compatibility risk.
- Suggest a focused Forge regression test when the issue is behavioral.
- If the fix is ambiguous, state the design decision needed instead of inventing one.

## Verification

When the review includes code changes, run the narrowest relevant `forge test` command first, then the full suite when feasible.

For read-only reviews, do not claim tests passed unless you ran them. Mention useful verification commands when they would raise confidence.

If RPC access, dependencies, environment variables, or fork configuration block verification, report the exact command and blocker.

## Output Contract

Default review output:

### Findings

Order findings by severity: `Critical`, `High`, `Medium`, `Low`, then non-blocking notes when useful.

For each finding, include:

- severity
- affected file and line or tight code location
- problem
- impact
- recommended fix
- suggested Forge test when useful
- confidence when useful

### Open Questions or Assumptions

List only questions that materially affect correctness or merge readiness.

### Summary

Briefly state reviewed files, verification performed, and residual risk.

If no material issues are found, say so explicitly and mention test gaps or surfaces not reviewed.

## Response Style

- Findings first
- Be concise and file-grounded
- Do not produce generic checklists
- Avoid praise-heavy review summaries
- Distinguish confirmed findings from recommendations
- Do not ask the user to inspect files you can inspect yourself
