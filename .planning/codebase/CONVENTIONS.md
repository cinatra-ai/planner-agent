# Coding Conventions

**Analysis Date:** 2026-06-09

## Naming Patterns

**Files:**
- Entry-point gate scripts use kebab-case with `.mjs` extension: `extension-kind-gate.mjs`
- Config sidecar files are placed under a dedicated `cinatra/` directory: `cinatra/oas.json`
- Skill content in `skills/<skill-name>/SKILL.md` uses kebab-case directory names

**Functions:**
- Exported validation functions use camelCase with a verb prefix describing action: `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `runGate`, `findWorkflowSidecars`, `parseArgs`
- Internal helpers use camelCase: `walkLlmStrings`, `scanOasString`, `wordBoundary`, `prefixOf`, `localOf`
- Main entry point is a plain `main()` function at the bottom of the file

**Variables:**
- Constants use SCREAMING_SNAKE_CASE: `LLM_VISIBLE_FIELDS`, `BANNED_PRIMITIVES`, `BANNED_TYPEHINTS`, `PRIMITIVE_PATTERNS`, `BPMN_MODEL_NS`, `WORKFLOW_PACKAGE_NAME_RE`
- Local variables use camelCase: `packageRoot`, `findings`, `errors`, `openTags`

**Types:**
- TypeScript strict mode is enabled (`"strict": true`) but `noImplicitAny` is explicitly relaxed to `false`
- Explicit `Error` type-narrowing used: `err instanceof Error ? err.message : String(err)`

## Code Style

**Formatting:**
- No Prettier or Biome config detected — formatting is not enforced by tooling in this repo
- Indentation: 2 spaces (observed throughout `extension-kind-gate.mjs`)
- Semicolons: present
- Double quotes for strings

**Linting:**
- No ESLint config detected
- TypeScript provides the primary static-analysis layer via `tsconfig.json`

## Import Organization

**Order:**
1. Node built-in modules (with `node:` prefix): `node:fs`, `node:path`
2. No third-party or internal imports present in this repo

**Path Aliases:**
- Not applicable — no alias configuration detected

## Error Handling

**Patterns:**
- All validation functions return `string[]` errors rather than throwing; callers collect and report
- `try/catch` wraps filesystem reads (`readFileSync`) with an error push into the `errors` array, then early-return
- `instanceof Error` guard used before accessing `.message`, with `String(err)` fallback: `err instanceof Error ? err.message : String(err)`
- Parse failures are returned as errors in the array, not thrown
- The gate main() function wraps `main()` in `try/catch` and calls `process.exit(1)` on unexpected errors

## Logging

**Framework:** `console.log` / `console.error` (native Node)

**Patterns:**
- Success output goes to `console.log` with a `✓` prefix
- Violation output goes to `console.error` with a `✗` prefix and bullet-point list `• item`
- Neutral/informational CI messages go to `console.log` without a prefix symbol

## Comments

**When to Comment:**
- File-level block comments describe the script's purpose, scope, and constraints (see `extension-kind-gate.mjs` lines 1–33)
- Section-level comments delimit logical blocks with dashed separator lines and a label
- Inline comments clarify non-obvious decisions, especially around skipped validations

**JSDoc/TSDoc:**
- JSDoc `/** ... */` used on exported functions to document purpose, inputs, and return contract
- Example: `validateAgent`, `validateBpmnSanity`, `findWorkflowSidecars`, `runGate` all carry JSDoc blocks

## Function Design

**Size:** Functions are small and focused; each exported function validates exactly one concern

**Parameters:** Functions accept either `packageRoot: string` (directory path) or a parsed data value (XML string, package object)

**Return Values:** Validation functions return `string[]` (empty = pass, non-empty = fail). The gate dispatcher `runGate` returns `{ kind, errors }`.

## Module Design

**Exports:** Named exports only — all public functions are explicitly `export function`. No default export.

**Barrel Files:** Not applicable — single-file module (`extension-kind-gate.mjs`)

**Module Format:** ESM (`"type": "module"` in `package.json`); all imports use `node:` protocol prefix for built-ins

## Cinatra Agent Skill Conventions

**Skill files:** `skills/<skill-name>/SKILL.md` contains YAML frontmatter with `name`, `description`, `match_when` directives followed by a markdown body

**Output contract:** Agent skill outputs are strict JSON objects (not bare arrays). The `findings` key is required even for empty results: `{"findings": []}` — bare arrays cause downstream WayFlow DataFlowEdge failures

**Severity:** All `ReviewFinding` objects emitted by this agent use `"severity": "suggestion"` only; hard blockers are owned by deterministic lint tools

---

*Convention analysis: 2026-06-09*
