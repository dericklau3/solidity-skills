---
name: develop-secure-contracts
description: "Use when integrating or extending OpenZeppelin Contracts in Solidity projects that compile with Solidity 0.8.30 or newer, including token standards, access control, security primitives, governance, and account components."
license: AGPL-3.0-only
metadata:
  author: OpenZeppelin
---

# Develop Secure Smart Contracts with OpenZeppelin

## Scope and Compatibility

This skill targets **Solidity 0.8.30 and newer only**.

Before making implementation changes:

1. Read the contract `pragma` statements.
2. Read the project's actual compiler configuration (`foundry.toml`, Hardhat config, or equivalent).
3. Confirm the compiler used by the project is Solidity `>=0.8.30`.

Do not treat a broad pragma range alone as proof that the project builds with 0.8.30+. If the configured compiler is older than 0.8.30, report the incompatibility and do not apply this skill's implementation or validation-style rules as authoritative for that project.

## Core Workflow

### Understand the Request Before Responding

For conceptual questions ("How does Ownable work?"), explain without generating code. For implementation requests within the supported compiler scope, proceed with the workflow below.

### CRITICAL: Always Read the Project First

Before generating code or suggesting changes:

1. **Search the user's project** for existing Solidity contracts (`Glob` for `**/*.sol`).
2. **Read the relevant contract files** to understand what already exists.
3. **Default to integration, not replacement** — when users say "add pausability" or "make it upgradeable", they mean modify their existing code, not generate something new. Only replace if explicitly requested ("start fresh", "replace this").

If a file cannot be read, surface the failure explicitly — report the path attempted and the reason. Never silently fall back to a generic response as if the file does not exist.

### Fundamental Rule: Prefer Library Components Over Custom Code

Before writing ANY logic, search the OpenZeppelin library for an existing component:

1. **Exact match exists?** Import and use it directly — inherit or compose with it. Done.
2. **Close match exists?** Import and extend it — override only functions the installed library marks as overridable (`virtual`, hooks, configurable parameters).
3. **No match exists?** Only then write custom logic. Confirm by browsing the installed library's directory structure first.

**NEVER copy or embed library source code into the user's contract.** Always import from the dependency so the project receives dependency updates. Never hand-write what the installed library already provides:

- Never write a custom `paused` modifier when `Pausable` or `ERC20Pausable` exists.
- Never write `require(msg.sender == owner)` when `Ownable` exists.
- Never implement ERC165 logic when the library's base contracts already handle it.

### CRITICAL: Validation Style — Prefer `require` with Custom Errors

For supported Solidity 0.8.30+ projects, ordinary precondition checks, validation, access/state guards, and invariant-style input checks should **prefer `require(condition, Errors.Xxx(...))`** over `if (!condition) revert Errors.Xxx(...);`.

Use a centralized `Errors` namespace/library when the project already has one. For new project-specific errors, prefer defining or reusing them in a shared `Errors.sol` rather than scattering error declarations across contracts, unless the existing project has a different explicit convention.

**Preferred:**

```solidity
require(account != address(0), Errors.ZeroAddress());
require(amount > 0, Errors.InvalidAmount());
require(balance >= amount, Errors.InsufficientBalance(balance, amount));
```

**Avoid for simple validation:**

```solidity
if (account == address(0)) revert Errors.ZeroAddress();
if (amount == 0) revert Errors.InvalidAmount();
if (balance < amount) revert Errors.InsufficientBalance(balance, amount);
```

Rules:

1. Write the condition in the **success form**: `require(validCondition, Errors.Xxx())`.
2. Reuse the project's existing `Errors.Xxx(...)` custom errors before creating new ones.
3. Do not replace custom errors with revert strings such as `require(x, "INVALID")` unless the user explicitly requests revert strings.
4. Do not use `if (...) revert ...` merely as a stylistic alternative to a simple `require` guard.
5. `if (...) revert ...` is acceptable when the revert belongs to genuinely branch-dependent control flow that cannot be expressed cleanly as a simple precondition, or when the user explicitly requests the `if/revert` form.
6. Remember that `require` arguments are evaluated unconditionally. Do not put calls with side effects or unnecessarily expensive computations inside `Errors.Xxx(...)` arguments.
7. When editing an existing function, convert newly touched simple `if (!condition) revert Errors.Xxx();` guards to the preferred `require(condition, Errors.Xxx());` form while keeping the change focused on the requested area.
8. Preserve an existing project convention when changing it would create broad style churn outside the requested scope.

### Methodology

The primary workflow is **pattern discovery from installed library source code**:

1. Inspect what the user's project already imports.
2. Read the dependency source and docs in the project's installed packages.
3. Identify what functions, modifiers, hooks, storage, constructors, or initializers the installed dependency requires.
4. Apply those requirements to the user's contract with the smallest compatible diff.

See [Pattern Discovery and Integration](#pattern-discovery-and-integration) below for the full procedure.

### CLI Generators as Reference

Use `npx @openzeppelin/contracts-cli` as a **reference implementation generator** when a relevant command exists: generate a baseline to a file, generate a feature-enabled variant to another file, diff them, then apply the relevant pattern to the user's code.

The CLI output is a reference, **not the final source of truth for an existing project**. The project's configured compiler and exact installed OpenZeppelin dependency source are authoritative for available APIs, inheritance, storage, hooks, constructor/initializer requirements, and override points.

If no CLI command exists for what's needed, use the generic pattern discovery methodology below. The absence of a CLI command does not mean the installed library lacks support.

## Pattern Discovery and Integration

Procedural guide for discovering and applying OpenZeppelin integration patterns by reading dependency source code for Solidity projects.

**Prerequisite:** Always follow the library-first decision tree above (prefer library components over custom code, never copy/embed source).

### Step 1: Identify Dependencies and Search the Library

1. Search the project for Solidity contract files: `Glob` for `**/*.sol`.
2. Read import statements in existing contracts to identify which OpenZeppelin components are already in use.
3. Locate the installed dependency in the project's dependency tree:
   - Hardhat/npm: `node_modules/@openzeppelin/contracts/`
   - Foundry/forge: `lib/openzeppelin-contracts/`
4. Browse the installed dependency's directory listing to discover available components. Use `Glob` patterns against the installed source (for example, `node_modules/@openzeppelin/contracts/**/*.sol`). Do not assume knowledge of the library's contents — verify from the installed version.
5. If the dependency is not installed locally, use the canonical OpenZeppelin Contracts repository only as a fallback reference and make the missing local-version evidence explicit.

### Step 2: Read the Dependency Source and Documentation

1. Read the source file of the component relevant to the user's request.
2. Look for documentation within the source: Solidity NatSpec comments (`///`, `/** */`) and README files in the component's directory.
3. Determine the integration strategy:
   - If the component satisfies the need directly → import and use as-is.
   - If customization is needed → identify extension points the installed library provides (`virtual` functions, hook functions, configurable constructor parameters). Import and extend.
   - Only if no component covers the need → write custom logic.
4. Identify the **public API**: functions exposed, events emitted, errors defined.
5. Identify **integration requirements**:
   - Functions the integrator MUST implement (abstract functions, overrides, hooks).
   - Modifiers or guards that must be applied to integrator functions.
   - Constructor or initializer parameters that must be passed.
   - Storage variables or state that must be declared.
   - Inheritance required (always via import, never via copy).
6. Search for examples or tests in the same installed dependency that demonstrate correct usage. Look in `test/`, `tests/`, `examples/`, or `mocks/` directories when available.

### Step 3: Extract the Minimal Integration Pattern

From Step 2, construct the minimal set of changes needed:

- **Imports** to add.
- **Inheritance** to add.
- **Storage** to declare.
- **Constructor / initializer** changes.
- **New functions** to add (required overrides, hooks, public API).
- **Existing functions to modify** (modifiers, hooks, events, integration calls).

If the contract is upgradeable, treat storage compatibility as a hard boundary. Inspect the existing storage layout, inheritance order, initializer/reinitializer behavior, namespaced storage or storage gaps, and upgrade authorization before applying changes. Do not alter storage compatibility casually or as an incidental side effect of adding an OpenZeppelin feature.

Do not include anything beyond what the installed dependency and requested behavior require. Aim for the minimal compatible diff between "contract without the feature" and "contract with the feature."

### Step 4: Apply Patterns to the User's Contract

1. Read the user's existing contract file.
2. Apply only the changes identified in Step 3. Do not replace the entire file when a focused integration is sufficient.
3. Check for conflicts: duplicate access-control systems, conflicting overrides, incompatible inheritance, initializer ordering, storage-layout effects, and duplicate security primitives.
4. Enforce the validation style above in newly touched simple guards when the project uses Solidity 0.8.30+ and doing so does not create unrelated churn.
5. Keep the edit focused on the requested feature.

### Repository and Documentation Lookup

| Ecosystem | Repository | Documentation | File Extension | Dependency Location |
| --- | --- | --- | --- | --- |
| Solidity | [openzeppelin-contracts](https://github.com/OpenZeppelin/openzeppelin-contracts) | [docs.openzeppelin.com/contracts](https://docs.openzeppelin.com/contracts) | `.sol` | `node_modules/@openzeppelin/contracts/` or `lib/openzeppelin-contracts/` |

### Directory Structure Conventions

Where to find components within the OpenZeppelin Contracts repository:

| Category | Solidity |
| --- | --- |
| Tokens | `contracts/token/{ERC20,ERC721,ERC1155}/` |
| Access control | `contracts/access/` |
| Governance | `contracts/governance/` |
| Proxies / Upgrades | `contracts/proxy/` |
| Utilities / Security | `contracts/utils/` |
| Accounts | `contracts/account/` |

Browse these paths first when searching for a component, but verify the actual installed directory layout before relying on a path.

### Known Version-Specific Considerations

Do not assume override points from prior knowledge — always verify by reading the installed source. Functions that were `virtual` in an older version may no longer be in the current one, and hooks can move or be replaced between major versions.

A known example is the Solidity ERC-20 transfer hook change between OpenZeppelin Contracts v4 and v5. Read the installed `ERC20.sol` before recommending or implementing an override.

## CLI Generators

The `@openzeppelin/contracts-cli` package can generate reference OpenZeppelin contract implementations from the command line. Use it as a pattern-discovery aid when a command exists for the relevant Solidity contract type.

### Discovering Commands and Options

Run `npx @openzeppelin/contracts-cli --help` to list available commands. For a Solidity contract command, run `npx @openzeppelin/contracts-cli <command> --help` to inspect current options. Do not rely on remembered flags because the CLI can change independently of the project's installed Contracts version.

### Generate-Compare-Apply Shortcut

When a CLI command exists for the contract type:

1. **Generate baseline** — run with only required options and pipe output to a temporary file.

   ```bash
   npx @openzeppelin/contracts-cli solidity-erc20 --name MyToken --symbol MTK > /tmp/oz-baseline.sol
   ```

2. **Generate with feature** — generate the requested feature variant.

   ```bash
   npx @openzeppelin/contracts-cli solidity-erc20 --name MyToken --symbol MTK --pausable > /tmp/oz-variant.sol
   ```

3. **Compare** — diff the two files to identify the integration shape.

   ```bash
   diff /tmp/oz-baseline.sol /tmp/oz-variant.sol
   ```

4. **Validate against the installed dependency** — verify every import, API, constructor/initializer, inheritance requirement, and override point against the project's exact installed source.
5. **Apply** — edit the user's existing contract with only the validated changes.

For interacting features, generate a combined variant as an additional reference when useful.

### When No CLI Command Exists or a Feature Is Not Covered

The absence of a CLI command does NOT mean the library lacks support. Fall back to installed-source pattern discovery for that feature. Read the exact dependency component, extract its integration requirements, and apply the smallest compatible change.