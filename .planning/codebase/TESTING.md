# Testing Patterns

**Analysis Date:** 2026-06-09

## Test Framework

**Runner:**
- Not detected — no test framework is declared in `package.json` (no `scripts.test`, no `jest`/`vitest`/`mocha` in dependencies)
- Config: none present

**Assertion Library:**
- Not detected

**Run Commands:**
```bash
# No test script defined in package.json
# CI runs: corepack pnpm test --if-present (exits 0 when no test script)
```

## Test File Organization

**Location:**
- No test files found in this repository (`*.test.*` / `*.spec.*` — none detected)

**Naming:**
- Not applicable

**Structure:**
- Not applicable

## Test Structure

**Suite Organization:**
- Not applicable — no tests exist in this repo

**Patterns:**
- Not applicable

## Mocking

**Framework:** Not applicable

**Patterns:**
- Not applicable

**What to Mock:**
- Not applicable

**What NOT to Mock:**
- Not applicable

## Fixtures and Factories

**Test Data:**
- Not applicable — no test infrastructure

**Location:**
- Not applicable

## Coverage

**Requirements:** None enforced

**View Coverage:**
```bash
# Not configured
```

## Test Types

**Unit Tests:**
- None present. The gate logic in `extension-kind-gate.mjs` is structured as pure functions (`validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate`) that take data in and return `string[]` errors — making them straightforward to unit test without mocks if tests were added.

**Integration Tests:**
- None present.

**E2E Tests:**
- Not used.

## CI Validation (Substitute for Tests)

The primary correctness gate is **`extension-kind-gate.mjs`** run during CI via GitHub Actions (`.github/workflows/ci.yml`). It validates:

1. `cinatra/oas.json` parses as valid JSON
2. No retired CRM primitives appear in LLM-visible OAS prompt strings (`system`, `user`, `description` fields)
3. No banned entity typeHints (`@cinatra-ai/entity-accounts:contact`, etc.) in OAS strings
4. No `objects_list` over CRM entity types in OAS strings

**CI Jobs:**
- `build` job — runs on Node 24, validates first-party dep shape, installs (if standalone), typechecks, runs tests if present, does `npm pack --dry-run`
- `kind-gates` job (needs `build`) — runs `node extension-kind-gate.mjs --package-root .`

**Typecheck:**
- The `tsconfig.json` targets `src/**/*.ts` / `src/**/*.tsx` but no `src/` directory currently exists; typecheck is effectively skipped for this content-only extension repo (CI skips `tsc` when no tracked TS files are found)

## Notes for Adding Tests

If tests are added, the pure-function design of `extension-kind-gate.mjs` makes it ideal for direct unit testing:
- Each exported function (`validateAgent`, `validateBpmnSanity`, `validateWorkflowPackageShape`, etc.) takes a string or object and returns `string[]` — no I/O mocking needed for most paths
- Only `findWorkflowSidecars` and `validateWorkflow`/`validateAgent` at the filesystem-read level require temp directories or mocking `readFileSync`
- Place test files adjacent to the source or in a `__tests__/` directory
- Recommended runner: Vitest (ESM-native, matches `"type": "module"` package)

---

*Testing analysis: 2026-06-09*
