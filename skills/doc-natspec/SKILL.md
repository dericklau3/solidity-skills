---
name: doc-natspec
description: "Use when reviewing, completing, or repairing NatSpec for Solidity / Foundry contracts after development, including public APIs, meaningful internal behavior, structs, events, overrides, and inherited documentation."
license: AGPL-3.0-only
metadata:
  author: derick
---

# Write and Repair NatSpec for Solidity Contracts

## Core Workflow

### Understand the Request Before Editing

Use this skill after Solidity development is complete and the user wants NatSpec coverage repaired, completed, or checked. By default, inspect contracts under the Foundry project's `src/` directory unless the user specifies exact files.

### Read the Project First

Before editing NatSpec:

1. Read `foundry.toml` when it exists.
2. Search `src/` for the relevant Solidity contracts.
3. Inspect project-owned interfaces that define the API of in-scope implementations, even when those interfaces are not themselves being edited.
4. Exclude libraries from the default edit scope unless the user explicitly asks to document them.
5. Read adjacent tests and supporting contracts when needed to resolve semantics.
6. Read `remappings.txt`, `lib/`, `node_modules/`, or package config when inherited behavior, overrides, initializers, or imported types determine meaning.

If a required file cannot be read, say so explicitly and do not pretend the documentation pass is complete.

### Default Scope

By default, inspect:

- Solidity contracts under `src/`.
- Project-owned interfaces that define the public API of those contracts.
- User-specified Solidity files.

By default, edit:

- In-scope implementation contracts.
- Project-owned interfaces only when their documentation is missing, inaccurate, or the user explicitly asks to document interfaces.

By default, do not edit:

- Vendored dependency interfaces or libraries.
- Pure test files.
- Unrelated historical files.

### Required Coverage

NatSpec is required for:

- every `public` function in scope that does not appropriately inherit sufficient documentation;
- every `external` function in scope that does not appropriately inherit sufficient documentation;
- every complex `internal` function in scope;
- every complex `private` function in scope;
- every externally meaningful `struct` in scope;
- every externally meaningful `event` in scope.

Tiny helpers, obvious passthrough wrappers, and trivial internal plumbing do not need forced NatSpec unless the user explicitly requests exhaustive internal coverage.

### Quality Standard

For standalone functions or functions whose behavior is not sufficiently documented by inheritance:

- require `@notice`;
- require `@dev` when behavior, restrictions, permissions, side effects, invariants, or edge cases are non-obvious;
- require one `@param` per input parameter;
- require one `@return` per return value;
- ensure the text matches the implementation;
- reject low-information filler that only restates the identifier;
- avoid documenting behavior the implementation does not guarantee.

For `struct` declarations:

- explain what the struct represents;
- explain important fields individually;
- focus on field meaning instead of repeating the type.

For `event` declarations:

- explain when and why the event is emitted;
- explain important parameters individually;
- make the event understandable for off-chain observers and integrators.

Comments must be accurate, specific, and useful to both documentation readers and reviewers. Passing `forge doc` is the minimum verification bar, not the quality bar.

### Inheritance and `@inheritdoc`

Do not mechanically duplicate inherited documentation.

When an override preserves the documented behavior of a project-owned interface or base contract:

- prefer inherited NatSpec or `@inheritdoc` instead of copying the same `@notice`, `@param`, and `@return` text into the implementation;
- use `@inheritdoc BaseName` when an explicit inheritance reference improves clarity or is needed to identify which base declaration supplies the documentation;
- add local `@dev` or fuller local NatSpec only when the override changes, narrows, extends, or clarifies behavior that a caller or reviewer must understand.

When an override materially changes behavior, permissions, side effects, revert conditions, accounting, state transitions, or integration semantics, do not rely on inherited docs alone. Document the changed behavior explicitly.

Do not add `@inheritdoc` blindly. First inspect the inherited declaration and confirm that its documentation is correct and sufficient for the implementation being documented.

### Dependency and Inheritance Rules

When documenting contracts that inherit from or compose OpenZeppelin or other dependencies:

- Read the installed dependency source before documenting overrides, hooks, modifiers, initializers, inherited storage, or extension behavior.
- Do not duplicate imported library NatSpec unless the user's contract changes the behavior or integration semantics.
- For project-owned interfaces and base contracts, inspect their NatSpec as part of understanding the public API even if they are outside the default edit scope.
- Document why a user-defined override exists when that reason is not obvious from the inherited API, what inherited requirement it satisfies, and what assumptions it preserves.
- For upgradeable contracts, document initializer order, reinitializer intent, storage-layout assumptions, and externally meaningful upgrade restrictions when they are part of the contract's public integration surface.
- Do not claim guarantees from a library component unless the installed source actually provides them.

## Editing Rules

### Patch Clear Gaps Directly

If the semantics are local and clear from code, naming, tests, inherited declarations, and call paths, directly add or improve the NatSpec comments.

### Prefer Inheritance Over Duplication

If a function already receives complete and accurate inherited NatSpec, do not add redundant local comments merely to satisfy tag-count expectations. Documentation quality and accuracy take priority over duplicated coverage.

### Stop on Ambiguity

If the meaning of a function, field, event parameter, override, or inherited guarantee cannot be established reliably, do not invent documentation. Call out the exact object, explain the ambiguity, and leave it unchanged.

### Keep Scope Tight

Do not turn a NatSpec pass into a refactor, rename sweep, or repository-wide style rewrite.

## Verification

After edits, run `forge doc`.

If it fails, explain whether the failure appears caused by:

- the new NatSpec edits;
- a pre-existing compile or configuration issue;
- a broader project problem unrelated to the current documentation patch.

Do not claim the work is complete without reporting the `forge doc` result.

## Output Contract

Default final output should include:

- reviewed files;
- whether project-owned interfaces or inherited declarations were inspected;
- `forge doc` result;
- any remaining documentation risks or unclear semantics.

## Response Style

- Be direct.
- Stay file-grounded.
- Do not fabricate meaning.
- Prefer accurate inherited documentation over duplicated comments.
- Do not confuse minimum tag coverage with good documentation.
