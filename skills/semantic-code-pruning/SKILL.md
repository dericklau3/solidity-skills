---
name: semantic-code-pruning
description: "Use when reviewing or editing code that looks bloated, redundant, over-defensive, or AI-generated, especially dead assignments, unused returns, duplicated guards, pass-through temporaries, thin wrappers, unreachable branches, and semantic no-ops."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Semantic Code Pruning

## Purpose

Use this skill to remove code that adds no observable behavior. This is not a style pass, minification pass, or general refactor. It is a semantic-preservation pass: every deletion must be backed by evidence that the program behaves the same for all supported callers and inputs.

Prefer a smaller change that is provably behavior-preserving over a clever rewrite that merely looks cleaner.

## Pruning Standard

A line, branch, helper, return value, or guard is removable only when all three conditions hold:

1. Its result is already guaranteed by surrounding control flow, types, state, invariants, caller contracts, or earlier validation.
2. Removing it does not change externally observable behavior, including return values, errors, logs, emitted events, database writes, network calls, storage layout, timing-sensitive side effects, metrics, or public API shapes.
3. The proof can be explained from code evidence, not from taste, naming, or the fact that tests do not currently cover the case.

If the proof depends on assumptions outside the repository, configuration, runtime environment, or public contract, treat the change as suggest-only unless the user confirms those assumptions.

## Evidence To Gather

Before editing, inspect enough context to prove the redundancy:

- Call graph: who can reach this code, with which arguments, and through which validation path.
- Data flow: where the value is assigned, transformed, observed, returned, persisted, or passed to side-effecting code.
- Control flow: whether earlier branches, exceptions, guards, pattern matches, or state-machine transitions make the code unreachable or duplicate.
- Contracts and interfaces: whether signatures, overrides, serialization formats, schemas, ABI/API compatibility, or generated clients depend on the current shape.
- Side effects: whether the code touches IO, storage, logs, telemetry, locks, caches, randomness, time, external calls, callbacks, or lifecycle hooks.
- Test and runtime evidence: which focused tests, builds, type checks, snapshots, or traces can confirm the behavior after pruning.

When the project has a domain-specific framework, respect its hidden contracts. Examples include React hook rules, database migration ordering, Solidity upgradeable storage layout, protocol event shapes, public SDK APIs, CLI output, and generated-code boundaries.

## What Usually Can Be Removed

| Candidate | Delete only when |
|---|---|
| Redundant assignment | Every reaching path already gives the variable the same value, and no observer depends on the write itself. |
| Unused return value | No caller consumes it, it is not required by an interface/override/protocol, and removing it does not change public API shape. |
| Duplicated guard | The same condition has already been enforced on every path, with equivalent error semantics and no intentionally different diagnostic message. |
| Pass-through temporary | The variable is assigned once, read once, has no debugging or documentation value, and inlining does not obscure domain meaning. |
| Dead branch | The branch is unreachable under current preconditions, state-machine rules, type constraints, or exhaustive matching, and is not intentionally defensive for external input or future states. |
| Thin wrapper | The helper only renames one call, adds no invariant, boundary, logging, retry, authorization, normalization, or domain vocabulary. |
| Redundant conversion | The conversion cannot change representation, precision, ownership, encoding, or validation state. |
| Repeated calculation | Reusing or deleting it does not affect evaluation order, overflow/rounding, lazy execution, memoization, or side effects. |

## What To Preserve

Do not delete code merely because it looks noisy. Preserve code when it carries any of these responsibilities:

- Public contract: API/ABI shape, override compatibility, serialized output, CLI output, event/log format, error type, or revert/error selector.
- Safety boundary: validation of untrusted input, permissions, asset movement, auth/session checks, reentrancy/locking, concurrency coordination, or capability isolation.
- Operational boundary: logging, metrics, audit trails, tracing, cleanup, rollback, retries, rate limiting, cache invalidation, migration steps, or feature flags.
- Domain meaning: names or wrappers that encode business language, compliance rules, accounting categories, protocol states, or team conventions.
- Future-proofing with a real contract: guards for documented configuration ranges, version skew, external integrations, upgrade paths, or data migrations.
- Generated or framework-owned code: code that must match a schema, code generator, lifecycle hook, reflection system, or framework convention.

If preserving the code is correct but it looks suspicious, consider adding or improving a narrow comment only when the invariant is non-obvious and the repository style supports such comments.

## Workflow

1. Identify the suspected no-op and classify its type: assignment, branch, guard, temporary, helper, return value, conversion, or repeated calculation.
2. Prove why it cannot affect behavior by citing the relevant caller path, invariant, type rule, state transition, interface contract, or side-effect analysis.
3. Check compatibility boundaries: public API, ABI, storage/schema layout, events/logs, error behavior, generated clients, migrations, and tests.
4. Apply the smallest deletion or simplification that preserves behavior. Avoid opportunistic rewrites nearby unless they are required by the pruning.
5. Run the narrowest meaningful verification first, then broader checks when the touched code is shared or externally visible.
6. Report what was removed, why it was safe, what was intentionally preserved, and which verification passed or failed.

## Risk Levels

- Safe to edit directly: private implementation detail, no side effects, proof is local and complete, tests or type checks cover the affected behavior.
- Edit only with strong verification: shared helper, public-facing behavior through indirect callers, framework lifecycle code, concurrency-sensitive paths, numeric/accounting logic, or serialization boundaries.
- Suggest only: public API/ABI changes, storage/schema layout changes, migration deletion, security guard removal, audit/logging removal, compatibility behavior, or changes that depend on undocumented assumptions.

When in doubt, leave the code in place and explain the uncertainty. A correct non-deletion is better than a tidy regression.

## Domain Notes

For Solidity and Foundry projects, preserve external/public function signatures, event shapes, custom error selectors, storage variable order, initializer arguments, modifiers, access control, upgradeable storage layout, and revert behavior unless the user explicitly requests a broader compatibility change. Keep guards around external input, asset movement, oracle/pair assumptions, authority checks, callbacks, and reentrancy boundaries unless the invariant is proven across all reachable entrypoints. Prefer `forge fmt --check`, `forge build`, and focused `forge test` for touched behavior.

For TypeScript, JavaScript, Go, Python, and similar application code, be careful with reflection, serialization, decorators, dynamic imports, framework lifecycle functions, dependency injection, generated types, public exports, and logging/telemetry that may be consumed outside the immediate code path.

## Output Contract

For each non-trivial pruning change, include:

- Removed code: the file/function and the kind of redundancy.
- Proof: the invariant or caller contract that makes it redundant.
- Boundary check: the API, side-effect, compatibility, or framework boundary considered.
- Verification: the exact checks run and their result.
- Deferred items: suspicious code left unchanged because the proof was incomplete or the risk was too high.

Keep the final answer concise, but do not omit the proof for behavior-sensitive deletions.
