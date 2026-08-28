---
name: semantic-code-pruning
description: "Use when reviewing or editing code that looks bloated, redundant, over-defensive, or AI-generated, especially dead assignments, unused returns, duplicated guards, pass-through temporaries, thin wrappers, unreachable branches, and semantic no-ops."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Prune Semantic No-Op Code

## Core Workflow

### Understand the Request Before Editing

Use this skill when the user wants code made smaller by removing redundant, dead, over-defensive, or AI-boilerplate logic.

This skill is for behavior-preserving semantic pruning. It is not a style sweep, formatter pass, broad refactor, minification pass, or speculative architecture cleanup.

### Read the Project First

Before making pruning edits:

1. Read the files in scope and the nearest README, package, build, or framework configuration when they explain runtime behavior.
2. Search for every caller, override, implementation, test, fixture, script, generated client, and public export that may observe the code being pruned.
3. Trace the relevant data flow from assignment to observation: return value, persisted state, emitted/logged output, external call, UI render, API response, or test assertion.
4. Trace the relevant control flow: guards, modifiers, earlier validation, exception paths, pattern matches, state-machine transitions, and framework lifecycle hooks.
5. Check whether public contracts depend on the current shape: ABI/API signatures, schemas, serialized fields, event or log format, error type, CLI output, migrations, or generated code.
6. Identify the strongest existing verification path: focused tests, type checks, build, snapshots, traces, gas reports, or framework-specific checks.

If a required caller, contract, schema, generated artifact, or verification path cannot be inspected, say so explicitly and do not treat the pruning as fully proven.

### Default Scope

By default, inspect only the files or behavior the user mentions.

If the user asks for a general pruning pass, inspect application or contract source first, then nearby tests and public interfaces. Exclude vendored dependencies, generated code, migrations, lockfiles, snapshots, and formatting-only churn unless the user explicitly includes them.

### Pruning Categories

Review opportunities across:

- dead assignments and overwritten values
- unused return values and unused outputs
- duplicated guards or already-enforced checks
- pass-through temporaries with no readability or debugging value
- unreachable branches under proven preconditions or state transitions
- thin wrappers that add no boundary, invariant, retry, logging, authorization, normalization, or domain vocabulary
- redundant conversions, repeated calculations, or no-op normalizations
- over-defensive branches introduced by AI-generated code without a reachable purpose

Do not prune mechanically. A line is removable only when its absence preserves behavior for every supported caller and input.

## Pruning Tiers

### 1. Safe Direct Edits

Directly patch low-risk pruning when the proof is local and complete.

Common examples:

- deleting an assignment that is overwritten before any read
- inlining a variable assigned once and read once when the name carries no domain meaning
- removing an unreachable private branch after all reaching callers enforce the same precondition
- removing a private return value when every caller ignores it and no interface requires it
- deleting a wrapper that only calls one private helper and adds no invariant, side effect, or useful vocabulary

Only patch these when no public contract, side effect, diagnostic behavior, or framework convention depends on the current code.

### 2. Conditional Edits

Patch only when the behavior is clear and verification is strong enough.

Common examples:

- merging duplicated branches with equivalent side effects and error behavior
- deleting defensive checks around internal state after every mutation path is proven
- simplifying repeated parsing, serialization, or normalization code
- pruning compatibility branches for documented, unsupported versions or modes
- removing redundant numeric calculations where rounding, overflow, ordering, and precision are fully understood

Only patch these when:

- the invariant is proven from code, not inferred from naming or current test coverage
- error types, messages, logs, events, metrics, and evaluation order remain equivalent where they are observable
- public API, schema, storage, migration, generated-client, and framework lifecycle boundaries are not changed
- available tests or checks cover the affected behavior strongly enough

### 3. Suggest-Only Edits

Leave these as recommendations unless the user explicitly approves a broader compatibility or cleanup change.

Common examples:

- changing public API, ABI, CLI output, serialized fields, or database schemas
- deleting migrations, compatibility adapters, feature flags, audit logs, telemetry, or rollback/cleanup code
- removing validation for untrusted input, authorization, asset movement, concurrency, locking, callbacks, or reentrancy
- pruning generated code or code that must match a schema, reflection system, decorator, dependency injection container, or lifecycle hook
- deleting future-proofing that protects documented configuration ranges, version skew, or external integrations
- removing code only because it is unused in the current tests

When the proof is incomplete, preserve the code and explain exactly what evidence is missing.

## Editing Rules

### Prove Before Patching

For each non-trivial deletion, first prove why the code cannot affect observable behavior. Use caller paths, data flow, control flow, type constraints, state invariants, interface contracts, and side-effect analysis as evidence.

### Patch The Smallest Proven Change

Delete only the redundant code. Do not opportunistically rewrite nearby logic, rename symbols, rearrange files, or change formatting outside the touched lines unless the pruning requires it.

### Preserve Observable Boundaries

Treat the following as pruning boundaries:

- public APIs, ABIs, exports, overrides, schemas, serialized output, CLI output, generated clients
- storage or database layout, migration history, event/log/error shapes, metrics, tracing, audit trails
- input validation, access control, auth/session checks, asset movement, locks, concurrency, cleanup, rollback
- framework lifecycle hooks, reflection, decorators, dependency injection, dynamic imports, generated code
- domain names that encode business meaning, accounting categories, protocol states, or team conventions

If code looks noisy but protects one of these boundaries, leave it in place or suggest a clearer comment instead of deleting it.

### Stop On Ambiguity

If deletion could change behavior, compatibility, diagnostics, execution order, persistence, or observability in a non-trivial way, do not guess. Name the suspicious code, explain the risk, and leave it as a recommendation.

### Keep Scope Tight

Do not turn a pruning pass into a broad refactor, security audit, optimization campaign, formatting sweep, or test rewrite. Route those separately when the user asks for them.

## Domain-Specific Boundaries

For Solidity and Foundry projects, preserve external/public function signatures, event shapes, custom error selectors, storage variable order, initializer arguments, modifiers, access control, upgradeable storage layout, and revert behavior unless the user explicitly requests a broader compatibility change. Keep guards around external input, asset movement, oracle/pair assumptions, authority checks, callbacks, and reentrancy boundaries unless the invariant is proven across all reachable entrypoints.

For TypeScript, JavaScript, Go, Python, and application code, be careful with reflection, serialization, decorators, dynamic imports, framework lifecycle functions, dependency injection, generated types, public exports, and logging or telemetry that may be consumed outside the immediate code path.

## Verification

Use the strongest existing project-specific verification path first.

Priority:

1. focused tests or regression tests for the touched behavior
2. type checks, linters, or framework checks that validate the touched contract
3. builds, snapshots, generated-client checks, schema checks, or trace comparisons
4. broader test suites when the touched code is shared or externally visible

For Solidity and Foundry projects, prefer touched-file `forge fmt --check`, `forge build`, and focused `forge test` for affected flows.

If verification fails, explain whether the failure appears caused by:

- the pruning change
- a pre-existing repository issue
- project configuration or dependency problems unrelated to the pruning patch

Do not claim the pruning is complete without reporting the verification result.

## Output Contract

Default final output should include:

- reviewed files and related callers
- pruning scope
- implemented deletions or simplifications
- proof that each non-trivial deletion preserves behavior
- suspicious code intentionally left unchanged
- risk notes
- verification result

Separate the pruning summary into:

- removed semantic no-ops
- preserved boundaries
- deferred recommendations

For skipped items, include the reason:

- proof incomplete
- public contract risk
- safety or operational boundary
- generated or framework-owned code
- compatibility or migration risk
- insufficient verification confidence

## Response Style

- Be direct
- Stay file-grounded
- Optimize for behavior-preserving deletion, not clever compression
- Do not cite tests alone as proof of redundancy
- Do not blur the boundary between pruning, optimization, security review, and refactoring
