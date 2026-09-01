---
name: foundry-post-dev-optimization
description: "Use when Solidity or Foundry contract development is complete and a post-dev optimization pass is needed for gas, semantic no-op pruning, code structure, or maintainability without turning the work into a security audit or full refactor."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Optimize Solidity Contracts After Development

## Core Workflow

### Understand the Request Before Editing

Use this skill after Solidity / Foundry development is complete and the user wants a focused optimization pass over existing contracts.

This skill covers post-development gas optimization and behavior-preserving semantic pruning together. Treat redundant variables, dead branches, thin wrappers, and repeated calculations as optimization work when their removal improves gas, code structure, or maintainability.

This skill is not for initial implementation, style sweeps, broad refactors, minification, security audits, or speculative architecture cleanup.

### Read the Project First

Before making optimization or pruning edits:

1. Read `foundry.toml` when it exists.
2. Search `src/` for Solidity contracts in scope.
3. Exclude interfaces unless the user explicitly asks to include them.
4. Read adjacent tests, mocks, fixtures, scripts, and helper contracts when needed to understand intended behavior.
5. Read `remappings.txt`, `lib/`, `node_modules/`, and package config when imports, inheritance, or library versions affect the optimization.
6. Search for every caller, override, implementation, test, fixture, script, generated client, and public export that may observe code being changed or pruned.
7. Trace relevant data flow from assignment to observation: return value, persisted state, emitted/logged output, external call, script output, ABI encoding, or test assertion.
8. Trace relevant control flow: guards, modifiers, earlier validation, revert paths, state-machine transitions, hooks, callbacks, and external calls.
9. Check whether public contracts depend on the current shape: ABI signatures, event fields, custom error selectors, revert behavior, storage layout, script arguments, generated artifacts, or deployment assumptions.
10. Check whether the repository already has a benchmark, snapshot, gas-report, or other optimization verification flow.

If a required caller, contract, dependency, generated artifact, storage-layout reference, or verification path cannot be inspected, say so explicitly and do not treat the optimization as fully proven.

### Default Scope

By default, inspect:

- Solidity contracts under `src/`

By default, exclude:

- interfaces
- vendored dependencies
- generated code
- migrations and deployment metadata
- lockfiles, snapshots, and formatting-only churn

If the user specifies particular contracts or files, narrow scope accordingly.

### Optimization Categories

Review opportunities across:

- gas efficiency
- semantic no-op pruning
- code structure
- maintainability

Do not optimize mechanically. A change is acceptable only when it preserves intended behavior for every supported caller and input, and the value is real enough to justify the edit.

### Semantic Pruning Categories

Actively look for behavior-preserving deletions and simplifications:

- dead assignments and overwritten values
- unused return values and unused outputs
- duplicated guards or already-enforced checks
- pass-through temporaries with no readability or debugging value
- private or internal helper parameters that only pass through storage references or values already recoverable from a canonical key argument
- unreachable branches under proven preconditions or state transitions
- thin wrappers that add no boundary, invariant, retry, logging, authorization, normalization, domain vocabulary, or gas benefit
- redundant conversions, repeated calculations, or no-op normalizations
- over-defensive branches introduced by AI-generated code without a reachable purpose

Do not prune because code "looks unused." Prune only after caller, data-flow, control-flow, type, state-invariant, interface-contract, and side-effect evidence proves the code cannot affect observable behavior.

### Library and Version Rule

Before replacing, simplifying, or micro-optimizing logic that overlaps with OpenZeppelin, Solady, token standards, proxy utilities, or other dependencies:

- Locate and read the installed dependency source. Do not assume an API, hook, or storage pattern from memory.
- Prefer importing, configuring, or extending a proven component over maintaining custom duplicated logic.
- Never paste dependency source into the user's contract as an optimization.
- Treat inherited hooks, required overrides, initializer order, namespaced storage, storage gaps, and state-variable order as optimization boundaries.
- If a suggested optimization could affect storage layout, ABI, event semantics, access control, or external integration assumptions, leave it as a recommendation unless the user explicitly approves the broader change.

## Optimization Tiers

### 1. Safe Direct Edits

Directly patch low-risk optimizations when semantics and proof are local and complete.

Common examples:

- `memory` to `calldata` where safe
- caching repeated storage reads or lengths
- deleting an assignment that is overwritten before any read
- inlining a variable assigned once and read once when the name carries no domain meaning
- removing an unreachable private branch after all reaching callers enforce the same precondition
- removing a private return value when every caller ignores it and no interface requires it
- removing redundant calculations, variables, or branches
- removing a private helper storage-reference parameter when the helper already receives the canonical mapping key and every caller passes `mapping[key]` from that same key
- deleting a wrapper that only calls one private helper and adds no invariant, side effect, or useful vocabulary
- replacing local duplicate utility logic with an already-installed, well-scoped library component when the behavior is identical

Only patch these when no public contract, side effect, diagnostic behavior, storage expectation, or framework convention depends on the current code.

### 2. Conditional Edits

Patch only when behavior is clear and verification is strong enough.

Common examples:

- custom errors replacing revert strings
- `unchecked`
- storage packing
- loop optimization
- state-variable reordering
- reducing repeated reads around state transitions and external calls
- merging duplicated branches with equivalent side effects and error behavior
- deleting defensive checks around internal state after every mutation path is proven
- simplifying repeated parsing, serialization, or normalization code
- pruning compatibility branches for documented, unsupported versions or modes
- changing inheritance, modifiers, or hooks to use a dependency-provided extension

Only patch these when:

- semantics are clear from code and context
- the invariant is proven from code, not inferred from naming or current test coverage
- error types, messages, logs, events, metrics, evaluation order, and revert behavior remain equivalent where observable
- no storage-layout or upgradeability assumption is violated
- no external integration expectation is broken
- available verification is strong enough to justify the change

### 3. Suggest-Only Edits

Leave these as recommendations unless the user explicitly wants a more aggressive optimization pass and the change is strongly justified.

Common examples:

- `assembly`
- ABI-shaping structural rewrites
- changing public ABI, event shape, custom error selector, CLI/script output, serialized fields, or generated artifacts
- deleting migrations, compatibility adapters, feature flags, audit logs, telemetry, rollback, or cleanup code
- removing validation for untrusted input, authorization, asset movement, oracle assumptions, callbacks, reentrancy, concurrency, or locking
- pruning generated code or code that must match a schema, reflection system, deployment script, decorator, dependency injection container, or lifecycle hook
- deleting future-proofing that protects documented configuration ranges, version skew, or external integrations
- low-value gas wins that materially reduce readability
- broad multi-contract refactors needed only for theoretical gains

Treat `assembly` more conservatively than other optimization techniques. When proof is incomplete, preserve the code and explain exactly what evidence is missing.

## Editing Rules

### Review Before Patching

Identify optimization and semantic-pruning opportunities before editing so the change set stays intentional and bounded.

### Prove Before Patching

For each non-trivial deletion or simplification, first prove why the code cannot affect observable behavior. Use caller paths, data flow, control flow, type constraints, state invariants, interface contracts, storage behavior, external-call ordering, and side-effect analysis as evidence.

For canonical-key helpers, prove every caller passes the same account, id, or key used to derive the storage reference, the helper does not intentionally operate on an alternate mapping entry, and internal lookup preserves delete, reload, and aliasing semantics. Keep explicit storage references when a caller must preserve a pre-fetched slot through complex mutations, operate on a different mapping entry, or avoid reloading around `delete` or reassignment behavior.

### Patch The Smallest Clear Win

Delete or rewrite only the code needed for the proven optimization. Do not opportunistically rename symbols, rearrange files, rewrite nearby logic, or change formatting outside touched lines unless the optimization requires it.

### Preserve Observable Boundaries

Treat the following as hard optimization boundaries unless the user explicitly approves a broader compatibility change:

- external/public function signatures, ABI, exports, overrides, generated clients, and script/deployment interfaces
- storage layout, state-variable order, initializer arguments, storage gaps, namespaced storage, migrations, and upgradeability assumptions
- event/log/error shapes, custom error selectors, revert behavior, metrics, tracing, and audit trails
- input validation, access control, auth/session checks, asset movement, oracle/pair assumptions, locks, callbacks, reentrancy, cleanup, and rollback
- framework lifecycle hooks, reflection, decorators, dependency injection, dynamic imports, and generated code
- domain names that encode business meaning, accounting categories, protocol states, or team conventions

If code looks noisy but protects one of these boundaries, leave it in place or suggest a clearer comment instead of deleting it.

### Stop on Ambiguity

If an optimization could change semantics, compatibility, diagnostics, execution order, persistence, storage expectations, integration assumptions, or observability in a non-trivial way, do not guess. Name the opportunity, explain the risk, and leave it as a recommendation.

### Keep Scope Tight

Do not turn an optimization pass into a large refactor, security audit, feature rewrite, style sweep, formatter pass, or test rewrite. Route those separately when the user asks for them.

## Solidity Patterns To Check

When reviewing Solidity loops, actively look for the low-value pattern:

```solidity
uint256 count;
for (...) {
    if (condition) count++;
}
T[] memory out = new T[](count);
for (...) {
    if (!condition) continue;
    out[index++] = value;
}
```

If the first loop only sizes memory for an event, return value, or local result, prefer a single loop that allocates the maximum input length, fills successful entries, then truncates the memory array length before emitting or returning. Keep all per-item validation, skip conditions, order, duplicate handling, and event payload semantics identical. Do not apply this when the count controls storage writes, authorization, pricing, external calls, gas-critical bounds, or any branch where over-allocation changes observable behavior.

## Verification

Use the strongest existing project-specific verification path first.

Priority:

1. existing benchmark scripts or CI performance baselines
2. existing gas snapshots or gas comparison workflows
3. existing gas-report workflow
4. focused tests or regression tests for the touched behavior
5. touched-file `forge fmt --check`
6. `forge build`
7. broader `forge test` when the touched code is shared or externally visible

If available, prefer before-and-after comparison over one-sided measurement. Do not fabricate gas improvements when only semantic cleanup was verified.

If verification fails, explain whether the failure appears caused by:

- the optimization or pruning change
- a pre-existing repository issue
- project configuration or dependency problems unrelated to the patch

Do not claim the optimization work is complete without reporting the verification result.

## Output Contract

Default final output should include:

- reviewed contracts and related files/callers
- optimization and pruning scope
- implemented gas changes
- implemented semantic no-op deletions or simplifications
- proof that each non-trivial deletion or simplification preserves behavior
- suggested but skipped optimizations
- suspicious code intentionally left unchanged
- risk notes
- verification result

Separate the optimization summary into:

- gas changes
- removed semantic no-ops
- code-structure changes
- maintainability changes
- preserved boundaries
- deferred recommendations

For skipped items, include the reason:

- semantics unclear
- proof incomplete
- public contract risk
- safety or operational boundary
- upgradeability or storage-layout risk
- generated or framework-owned code
- compatibility or migration risk
- requires aggressive `assembly`
- low-value gain not worth readability loss
- insufficient verification confidence

## Response Style

- Be direct
- Stay file-grounded
- Optimize for real value and behavior-preserving deletion, not clever compression
- Do not cite tests alone as proof of redundancy
- Do not fabricate gas improvements
- Do not blur the boundary between optimization, testing, security review, and broad refactoring
