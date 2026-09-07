---
name: develop-secure-contracts
description: "Use when integrating or extending OpenZeppelin Contracts in Solidity projects compiled with Solidity 0.8.30 or newer."
license: AGPL-3.0-only
metadata:
  author: OpenZeppelin
---

# Integrate OpenZeppelin Contracts

## Follow This Skill

Follow this skill's applicable scope, execution rules, verification requirements, and output contract. Do not silently skip required steps or substitute advice for authorized implementation. Explicit user instructions and higher-priority instructions take precedence. If a required step cannot be completed, state the specific limitation, continue independent work, and do not claim that step was completed.

## Scope

This skill targets Solidity projects compiled with **0.8.30 or newer**. Read pragmas and the actual compiler configuration; a broad pragma alone is insufficient. If the configured compiler is older, explain the mismatch and use guidance appropriate to that version instead of applying this skill's version-specific rules or silently upgrading the compiler.

Conceptual questions receive explanations. For implementation requests, modify the existing contracts directly within the authorized scope; replace them only when the user requests replacement. Explicit user instructions take precedence over skill defaults. Reuse prior authorization and resolve routine choices from the project.

## Discover the Integration

1. Locate in-scope `.sol` files with the available file-search tools, such as `rg --files -g '*.sol'`. Read target contracts, relevant tests, and compiler/package configuration. Resolve remappings and imports to the exact installed dependency, including `contracts-upgradeable` when used.
2. Read the relevant library component, its NatSpec, and useful local examples/tests. Verify APIs, `virtual` extension points, hooks, modifiers, inheritance, constructor/initializer arguments, and storage requirements from that version. Do not infer v5 hooks from v4 conventions.
3. Prefer an existing component that satisfies the request. Import and compose/inherit it, or extend only supported customization points. Write custom logic only when the installed library cannot provide the needed behavior. Do not copy library source into project contracts or duplicate existing access-control, pause, or interface-detection mechanisms.
4. Identify the smallest compatible set of imports, inheritance, overrides, guards, and initialization changes. Check conflicting overrides, duplicate security primitives, and integration effects on callers, events, errors, scripts, tests, and documentation.

If dependencies are missing, inspect the declared version/lockfile and use matching official source only as a disclosed fallback. An unresolved installed version is an evidence limit, not permission to assume the latest API. For unavailable files, report the path and cause, continue independent work, and ask only when missing information determines correctness.

## Apply the Change

Preserve unrelated work, business semantics outside the request, and project conventions. Apply the integration to existing code without a replacement scaffold or adjacent style sweep.

For upgradeable contracts, inspect the current deployment context, storage layout, inheritance order, namespaced storage/gaps, initializer/reinitializer ordering, and upgrade authorization. A new implementation for an existing proxy must remain compatible with that proxy's state. Validate against the deployed baseline or the project's upgrade validator before claiming compatibility; tests alone are insufficient. Missing baseline evidence must be reported as unverified.

### Validation Style for Solidity 0.8.30+

Prefer `require(validCondition, Errors.Xxx(...))` for new or substantively edited simple guards. Reuse a project `Errors` namespace; for new errors prefer a shared `Errors.sol` unless the project uses another convention.

```solidity
require(account != address(0), Errors.ZeroAddress());
require(balance >= amount, Errors.InsufficientBalance(balance, amount));
```

Preserve existing custom errors instead of introducing revert strings. Branch-dependent failure logic may retain `if (...) revert ...`; follow an explicit user style preference. Avoid touching unrelated guards solely to standardize syntax.

`require` evaluates its arguments unconditionally. Before converting an `if/revert` guard, inspect error-argument evaluation, including calls that may revert, have side effects, or incur meaningful cost. Retain conditional evaluation when conversion would change behavior or add unjustified work.

### Optional CLI Reference

Use `@openzeppelin/contracts-cli` only when generated examples would resolve an integration question. Inspect `--help` for the available command and flags; use a project-pinned version when present. If useful, generate baseline and feature variants into temporary files, compare them, and apply only the relevant changes verified against installed source. CLI output is not authoritative over the project's dependency version. Missing CLI support does not mean a library component is unavailable; continue source-based integration.

## Verification and Completion

Compile through the project's normal build/test command and run focused integration/regression tests for the changed behavior. Synchronize affected callers, scripts, tests, and documentation. Check formatting on touched files and inspect the diff.

For shared inheritance, initialization, permissions, accounting, or public behavior, cover all affected suites; use the full suite when the affected surface cannot be bounded. Honor explicit full-suite requests and required project checks. Once they pass, repeat or expand only for new changes, failures, or unresolved concerns. Report baseline failures and environment blockers separately from regressions.

Finish with the implemented behavior, installed-version evidence relevant to the change, actual verification, and any remaining compatibility or execution limit. Keep the response proportionate to the patch.
