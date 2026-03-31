---
name: doc-natspec
description: "Review and complete NatSpec comments for Solidity / Foundry contracts after development. Covers `public` and `external` functions, complex `internal` and `private` functions, and externally meaningful `struct` and `event` declarations, with `forge doc` verification."
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

1. Read `README.md`
2. Read `foundry.toml` when it exists
3. Search `src/` for the relevant Solidity contracts
4. Exclude interfaces and libraries unless the user explicitly asks to document them
5. Read adjacent tests and supporting contracts when needed to resolve semantics

If a required file cannot be read, say so explicitly and do not pretend the documentation pass is complete.

### Default Scope

By default, inspect:

- Solidity contracts under `src/`
- user-specified Solidity files

By default, do not inspect:

- interfaces
- libraries
- pure test files
- unrelated historical files

### Required Coverage

NatSpec is required for:

- every `public` function in scope
- every `external` function in scope
- every complex `internal` function in scope
- every complex `private` function in scope
- every externally meaningful `struct` in scope
- every externally meaningful `event` in scope

Tiny helpers, obvious passthrough wrappers, and trivial internal plumbing do not need forced NatSpec unless the user explicitly requests exhaustive internal coverage.

### Quality Standard

For functions:

- require `@notice`
- require `@dev` when behavior, restrictions, permissions, side effects, invariants, or edge cases are non-obvious
- require one `@param` per input parameter
- require one `@return` per return value
- ensure the text matches the implementation
- reject low-information filler that only restates the identifier
- avoid documenting behavior the implementation does not guarantee

For `struct` declarations:

- explain what the struct represents
- explain important fields individually
- focus on field meaning instead of repeating the type

For `event` declarations:

- explain when and why the event is emitted
- explain important parameters individually
- make the event understandable for off-chain observers and integrators

Comments must be accurate, specific, and useful to both documentation readers and reviewers. Passing `forge doc` is the minimum verification bar, not the quality bar.

## Editing Rules

### Patch Clear Gaps Directly

If the semantics are local and clear from code, naming, tests, and call paths, directly add or improve the NatSpec comments.

### Stop on Ambiguity

If the meaning of a function, field, or event parameter cannot be established reliably, do not invent documentation. Call out the exact object, explain the ambiguity, and leave it unchanged.

### Keep Scope Tight

Do not turn a NatSpec pass into a refactor, rename sweep, or repository-wide style rewrite.

## Verification

After edits, run `forge doc`.

If it fails, explain whether the failure appears caused by:

- the new NatSpec edits
- a pre-existing compile or configuration issue
- a broader project problem unrelated to the current documentation patch

Do not claim the work is complete without reporting the `forge doc` result.

## Output Contract

Default final output should include:

- reviewed files
- `forge doc` result
- any remaining documentation risks or unclear semantics

## Response Style

- Be direct
- Stay file-grounded
- Do not fabricate meaning
- Do not confuse minimum tag coverage with good documentation
