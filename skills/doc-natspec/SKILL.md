---
name: doc-natspec
description: "Use when reviewing, completing, or repairing Solidity / Foundry NatSpec, including public APIs, meaningful internal behavior, events, structs, and inherited documentation."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Write and Repair Solidity NatSpec

## Follow This Skill

Follow this skill's applicable scope, execution rules, verification requirements, and output contract. Do not silently skip required steps or substitute advice for authorized implementation. Explicit user instructions and higher-priority instructions take precedence. If a required step cannot be completed, state the specific limitation, continue independent work, and do not claim that step was completed.

## Scope and Execution

For documentation edits, directly repair clear gaps within the requested scope. Explicit review-only requests remain read-only. Otherwise use the configured source directory, normally `src/`. Inspect project-owned interfaces/base contracts defining the in-scope API; edit their docs only when missing, inaccurate, or explicitly requested.

Exclude vendored dependencies, pure tests, unrelated files, and libraries from default edits unless requested. Preserve implementation and unrelated work; do not turn documentation into a refactor or rename sweep. User instructions and existing authorization take precedence over skill defaults.

## Establish Meaning

Read compiler configuration, in-scope code, and relevant inherited declarations. Use adjacent tests and callers to resolve behavior. When dependency hooks, modifiers, initializers, storage, or imported types determine meaning, resolve remappings and read the exact installed source.

If a function, field, event parameter, or inherited guarantee remains ambiguous, leave that object's documentation unchanged, identify the uncertainty, and continue independent objects. Ask only when missing information materially determines the documentation and cannot be recovered from the project. Report unavailable files without treating the whole pass as blocked.

## Coverage and Quality

Require accurate NatSpec for in-scope public/external functions, complex internal/private functions, and externally meaningful structs/events. Tiny helpers and obvious passthroughs need no forced internal documentation.

For functions without sufficient inherited documentation:

- Include `@notice`, one `@param` per input, and one `@return` per return value.
- Include `@dev` for non-obvious permissions, restrictions, side effects, invariants, failure conditions, or edge cases.
- Explain actual guarantees and units; avoid filler that repeats identifiers or promises absent from the implementation.

For structs, explain their purpose and important fields. For events, explain when/why emission occurs and what parameters mean to off-chain consumers, including amounts/units and conditions where relevant.

## Inheritance

Reuse sufficient, accurate inherited documentation when an override preserves its semantics. Use `@inheritdoc BaseName` when it clarifies or selects the source. Do not duplicate tags merely to satisfy coverage counts.

Read the inherited declaration before relying on it. Add local documentation when an override changes, narrows, extends, or clarifies behavior, permissions, accounting, state transitions, side effects, reverts, or integration guarantees. Explain the purpose of a non-obvious override and the library requirement it satisfies.

For upgradeable contracts, document externally meaningful initializer order, reinitializer intent, storage assumptions, and upgrade restrictions. Do not claim dependency guarantees or verified layout compatibility without evidence.

## Verification and Final Response

After edits, run `forge doc` and inspect the diff for unintended implementation changes. The documentation build validates syntax/buildability, not semantic accuracy. Do not add unrelated tests for comment-only edits. Honor project-required checks; after they pass, repeat only for new changes, failures, or unresolved concerns.

If `forge doc` fails, distinguish new documentation errors from baseline compile/configuration issues or environment blockers. Report the actual command result and any unfinished coverage; do not imply a successful build.

Briefly summarize files documented, relevant interface/inheritance coverage, `forge doc` result, and remaining ambiguity. Omit empty categories and generic advice.
