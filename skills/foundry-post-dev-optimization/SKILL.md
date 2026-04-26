---
name: foundry-post-dev-optimization
description: "Use when Solidity or Foundry contract development is complete and a post-dev optimization pass is needed for gas, code structure, or maintainability without turning the work into a security audit or full refactor."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Optimize Solidity Contracts After Development

## Core Workflow

### Understand the Request Before Editing

Use this skill after Solidity / Foundry development is complete and the user wants a focused optimization pass over existing contracts. This skill is for post-development gas, code-structure, and maintainability optimization, not for initial implementation.

### Read the Project First

Before making optimization edits:

1. Read `foundry.toml` when it exists
2. Search `src/` for Solidity contracts in scope
3. Exclude interfaces unless the user explicitly asks to include them
4. Read adjacent tests, mocks, fixtures, scripts, and helper contracts when needed to understand intended behavior
5. Check whether the repository already has a benchmark, snapshot, gas-report, or other optimization verification flow

If a required file cannot be read, say so explicitly and do not pretend the optimization pass is complete.

### Default Scope

By default, inspect:

- Solidity contracts under `src/`

By default, exclude:

- interfaces

If the user specifies particular contracts or files, narrow scope accordingly.

### Optimization Categories

Review opportunities across:

- gas efficiency
- code structure
- maintainability

Do not optimize mechanically. Prefer meaningful, behavior-safe improvements over superficial changes.

## Optimization Tiers

### 1. Safe Direct Edits

Directly patch low-risk optimizations when semantics are clear.

Common examples:

- `memory` to `calldata` where safe
- revert strings to custom errors
- caching repeated storage reads or lengths
- removing redundant calculations, variables, or branches
- small helper extraction that improves clarity without changing behavior

### 2. Conditional Edits

Patch only when behavior is clear and verification is strong enough.

Common examples:

- `unchecked`
- storage packing
- loop optimization
- state-variable reordering
- reducing repeated reads around state transitions and external calls

Only patch these when:

- semantics are clear from code and context
- no storage-layout or upgradeability assumption is violated
- no external integration expectation is broken
- available verification is strong enough to justify the change

### 3. Suggest-Only Edits

Leave these as recommendations unless the user explicitly wants a more aggressive optimization pass and the change is strongly justified.

Common examples:

- `assembly`
- ABI-shaping structural rewrites
- low-value gas wins that materially reduce readability
- broad multi-contract refactors needed only for theoretical gains

Treat `assembly` more conservatively than other optimization techniques.

## Editing Rules

### Review Before Patching

Identify optimization opportunities before editing so the change set stays intentional and bounded.

### Patch Clear Wins Directly

If intended behavior is clear from the implementation, tests, naming, and repository context, directly apply safe or sufficiently justified optimizations.

### Stop on Ambiguity

If an optimization could change semantics, storage expectations, integration assumptions, or readability in a non-trivial way, do not guess. Name the opportunity, explain the risk, and leave it as a recommendation.

### Keep Scope Tight

Do not turn an optimization pass into a large refactor, security audit, feature rewrite, or style sweep.

## Verification

Use the strongest existing project-specific verification path first.

Priority:

1. existing benchmark scripts or CI performance baselines
2. existing gas snapshots or gas comparison workflows
3. existing gas-report workflow
4. fallback `forge test`

If available, prefer before-and-after comparison over one-sided measurement.

If verification fails, explain whether the failure appears caused by:

- the optimization changes
- a pre-existing repository issue
- project configuration or dependency problems unrelated to the optimization patch

Do not claim the optimization work is complete without reporting the verification result.

## Output Contract

Default final output should include:

- reviewed contracts and related files
- optimization scope
- implemented optimizations
- suggested but skipped optimizations
- risk notes
- verification result

Separate the optimization summary into:

- gas changes
- code-structure changes
- maintainability changes

For skipped items, include the reason:

- semantics unclear
- upgradeability or storage-layout risk
- requires aggressive `assembly`
- low-value gain not worth readability loss
- insufficient verification confidence

## Response Style

- Be direct
- Stay file-grounded
- Optimize for real value, not mechanical micro-optimizations
- Do not fabricate gas improvements
- Do not blur the boundary between optimization, testing, and security review
