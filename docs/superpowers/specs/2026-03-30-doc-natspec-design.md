# doc-natspec Skill Design

## Summary

Add a new `doc-natspec` skill under `skills/` for Solidity / Foundry projects. The skill is intended for post-development documentation cleanup: it should inspect the contracts touched by the recent work, identify missing or weak NatSpec comments, directly patch clear local gaps, and verify that generated docs still work with `forge doc`.

The skill is documentation-focused, not a general review or refactor skill. Its job is to make the public-facing contract surface complete, accurate, and readable for documentation consumers.

## Goals

- Provide a repeatable post-development workflow for NatSpec completion.
- Require high-quality NatSpec for all `public` and `external` functions in scope.
- Require NatSpec for externally meaningful `struct` and `event` declarations in scope.
- Ensure the resulting comments are compatible with `forge doc`.
- Keep edits scoped to the relevant contracts instead of turning the task into a repo-wide rewrite.

## Non-Goals

- Document every `internal` or `private` implementation detail.
- Refactor business logic, rename symbols, or rewrite unrelated contracts to improve docs.
- Invent semantic meaning that is not supported by code, tests, naming, or nearby context.
- Sweep the entire repository unless the user explicitly asks for that.

## Trigger and Scope

The skill is intended to be used after development work is complete.

Default scope rules:

- Inspect Solidity contracts under `src/`.
- Prioritize files touched by the current task, if they can be identified.
- Exclude interfaces and libraries by default.
- Include an interface or library only when the user explicitly asks for it.
- Exclude pure test files unless the user explicitly asks otherwise.

The skill may also operate on user-specified Solidity files even if they fall outside the default scope.

## Required Coverage

The skill must require NatSpec coverage for the following in-scope items:

- every `public` function
- every `external` function
- every externally meaningful `struct`
- every externally meaningful `event`

Coverage is mandatory for these objects when they are in scope. The skill should not treat `struct` and `event` documentation as optional polish.

## Quality Standard

Passing `forge doc` is necessary but not sufficient. The skill should enforce comments that are both machine-consumable and useful to human readers.

### Functions

For each in-scope `public` or `external` function:

- require `@notice`
- require `@dev` when behavior, restrictions, side effects, permissions, invariants, or edge cases are non-obvious
- require one `@param` per input parameter
- require one `@return` per return value
- ensure the description matches the current implementation
- reject empty or low-information wording that just paraphrases the identifier
- avoid documenting behavior the implementation does not guarantee

### Structs

For each in-scope `struct`:

- require a top-level description of what the struct represents
- explain important fields individually, especially amounts, addresses, timestamps, identifiers, indexes, state flags, and share/accounting values
- describe field meaning, not just type shape

### Events

For each in-scope `event`:

- require a top-level description of when and why the event is emitted
- explain important parameters individually, especially `indexed` fields, amounts, actors, identifiers, and before/after state values
- make the event understandable for off-chain observers and integrators

### Style Rules

- prefer concise, concrete wording
- keep terminology consistent within a contract
- document non-obvious constraints explicitly
- avoid filler descriptions such as restating the function name
- align comments to the actual semantics visible in code and tests

## Execution Flow

### 1. Build Context

The skill should read enough project context to document the code accurately:

- `README.md`
- `foundry.toml`, when present
- the target Solidity files
- adjacent tests or supporting contracts when needed to resolve meaning

### 2. Scan for NatSpec Gaps

Inspect in-scope declarations and identify:

- missing `@notice`
- missing or incomplete `@dev`
- missing `@param`
- missing `@return`
- undocumented `struct`
- undocumented `event`
- weak comments that are technically present but materially unhelpful

### 3. Decide Whether to Patch Automatically

If the missing documentation is local and the semantics are clear from the code and context, the skill should directly add or improve NatSpec comments.

If the missing documentation is broad, ambiguous, or dependent on information that cannot be reliably inferred, the skill should stop short of guessing and report the uncertainty.

### 4. Normalize Comment Quality

When patching comments, the skill should improve low-quality NatSpec in the touched scope rather than merely adding the minimum missing tags.

### 5. Verify with `forge doc`

After edits, the skill should run `forge doc` as the final verification step and report the result clearly.

## Ambiguity Handling

If the meaning of a function, field, or event parameter cannot be established reliably from:

- implementation
- naming
- call sites
- tests
- nearby comments
- user-provided context

then the skill must not invent the NatSpec. It should:

- name the specific object that remains unclear
- explain why the context is insufficient
- skip automatic patching for that object
- continue with other objects that are clear enough to document safely

## Output Contract

The skill's final response should include:

- which files were reviewed
- which functions, `struct`s, and `event`s were updated
- which declarations were intentionally left unchanged due to semantic ambiguity
- whether `forge doc` succeeded
- if `forge doc` failed, whether the failure appears caused by the new doc edits or by pre-existing project issues

## Risks and Guardrails

- Do not expand a documentation task into a business-logic refactor.
- Do not rewrite large unrelated areas just to achieve stylistic consistency.
- Do not document interfaces or libraries by default.
- Do not silently succeed without verification when `forge doc` is expected to run.

## Recommended Skill Structure

The skill should mirror the style of the existing repository skills:

- concise frontmatter with name, description, license, and metadata
- a direct workflow-oriented body
- explicit must / must-not guidance
- output expectations for the final response

## Open Decision Resolved

The following default behavior is part of the approved design:

- default operating mode is review first, then directly patch clear local NatSpec gaps
- default scan target is Foundry `src/` contracts
- interfaces and libraries are excluded unless explicitly requested by the user
