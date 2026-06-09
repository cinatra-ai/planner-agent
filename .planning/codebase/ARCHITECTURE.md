<!-- refreshed: 2026-06-09 -->
# Architecture

**Analysis Date:** 2026-06-09

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│              Cinatra OAS Flow 26.1.0 Agent                  │
│              @cinatra-ai/planner-agent                      │
└───────────────────────────┬─────────────────────────────────┘
                            │ invoked by agent_source_review
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  StartNode ("start")                                        │
│  Accepts: oasJson, packageSlug, reviewContext, agent_run_id │
└───────────────────────────┬─────────────────────────────────┘
                            │ ControlFlowEdge: start_to_review
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  ApiNode ("review") — POST /api/llm-bridge                  │
│  LLM: openai / gpt-5.5                                      │
│  Skill: design-review-methodology                           │
│  `cinatra/oas.json` ($referenced_components.review)         │
└───────────────────────────┬─────────────────────────────────┘
                            │ DataFlowEdge: review_to_end_findings
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  EndNode ("end")                                            │
│  Output: findings (JSON string)                             │
└─────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| StartNode (`start`) | Accepts flow inputs; `oasJson` required, others optional | `cinatra/oas.json` (`$referenced_components.start`) |
| ApiNode (`review`) | Calls `/api/llm-bridge` with LLM prompt; returns `{"findings":[...]}` | `cinatra/oas.json` (`$referenced_components.review`) |
| EndNode (`end`) | Exposes `findings` string as the flow output | `cinatra/oas.json` (`$referenced_components.end`) |
| Skill definition | Defines the LLM system prompt rules and review methodology | `skills/design-review-methodology/SKILL.md` |
| CI gate | Zero-dependency OAS validation: retired-primitive scan + package shape check | `extension-kind-gate.mjs` |

## Pattern Overview

**Overall:** Single-node LLM advisory pipeline — a linear `StartNode → ApiNode → EndNode` OAS Flow 26.1.0 graph with no branching, no HITL gate, and no sub-agent decomposition. The agent is advisory (design suggestions only); all hard blockers belong to deterministic linters in the broader `agent_source_review` orchestration.

**Key Characteristics:**
- Pure read-only advisory agent (`riskClass: "read_only"`, `requiresApproval: false`)
- Single executable step (one `ApiNode` — the design review itself)
- No MCP tool calls; all context delivered inline via `oasJson` input string
- Output is a JSON-in-string envelope `{"findings":[...]}` required by WayFlow's `DataFlowEdge` key extraction
- Skill (`design-review-methodology`) is loaded by the LLM runtime; not a runtime code import

## Layers

**Flow Definition Layer:**
- Purpose: Declares the agent topology, node types, inputs, outputs, and data/control connections in Cinatra OAS Flow 26.1.0 format
- Location: `cinatra/oas.json`
- Contains: StartNode, ApiNode, EndNode, ControlFlowEdge, DataFlowEdge, `$referenced_components`
- Depends on: Cinatra platform (`{{CINATRA_BASE_URL}}/api/llm-bridge`)
- Used by: Cinatra agent runner / `agent_source_review` orchestrator

**Skill / Methodology Layer:**
- Purpose: Provides the LLM with design-review rules, output contract, and what to check vs. what to skip
- Location: `skills/design-review-methodology/SKILL.md`
- Contains: Structured LLM system prompt (input spec, output contract, checks, anti-checks, steps)
- Depends on: Nothing (pure text delivered at LLM runtime)
- Used by: LLM model (gpt-5.5) within the `review` ApiNode

**CI Gate Layer:**
- Purpose: Self-contained, zero-dependency pre-publish sanity gate for extracted extension repos; runs in GitHub Actions without registry access
- Location: `extension-kind-gate.mjs`
- Contains: `validateAgent`, `validateWorkflow`, `validateBpmnSanity`, `validateWorkflowPackageShape`, `runGate`, `main`
- Depends on: Node.js builtins only (`node:fs`, `node:path`)
- Used by: `.github/workflows/ci.yml` (`kind-gates` job)

## Data Flow

### Primary Request Path

1. Orchestrator (`agent_source_review`) invokes `@cinatra-ai/planner-agent` with `oasJson`, `packageSlug`, `reviewContext`, `agent_run_id` (`cinatra/oas.json` → `start` node)
2. `StartNode` fans all four inputs out via `DataFlowEdge` to the `review` ApiNode (`cinatra/oas.json` → `data_flow_connections`)
3. `ApiNode` POSTs to `{{CINATRA_BASE_URL}}/api/llm-bridge` with an LLM prompt embedding `oasJson`, `packageSlug`, `reviewContext`; the skill SKILL.md shapes the system prompt (`cinatra/oas.json` → `$referenced_components.review`)
4. LLM returns `{"findings":[...]}` JSON object; DataFlowEdge extracts the `findings` key and routes it to EndNode
5. EndNode emits `findings` as the flow output string

**Output Contract:**
- Must always be `{"findings":[...]}` (object with `findings` array key); bare `[]` triggers `Cannot index array with string "findings"` in WayFlow's DataFlowEdge

## Key Abstractions

**ReviewFinding:**
- Purpose: A single design suggestion emitted by the agent
- Shape: `{ code, severity: "suggestion", message, location?, source: "agent-planner" }`
- Examples: node-choice suggestions, HITL placement advice, sub-agent decomposition hints
- Defined in: `skills/design-review-methodology/SKILL.md`

**OAS Flow 26.1.0 Graph:**
- Purpose: Declarative topology of the agent (nodes + edges)
- Pattern: `$referenced_components` map holds node definitions; `nodes`, `control_flow_connections`, `data_flow_connections` reference them via `$component_ref`
- File: `cinatra/oas.json`

**Extension Kind Gate:**
- Purpose: Per-kind CI validation dispatched by `cinatra.kind` field in `package.json`
- Pattern: `runGate(packageRoot)` → delegates to `validateAgent` (OAS retired-primitive scan) or `validateWorkflow` (BPMN shape check)
- File: `extension-kind-gate.mjs`

## Entry Points

**Agent Flow Entry:**
- Location: `cinatra/oas.json` → `start_node: { $component_ref: "start" }`
- Triggers: Cinatra platform dispatches this agent (typically from `agent_source_review` when OAS is non-trivial)
- Responsibilities: Receives inputs, routes to LLM review node, surfaces findings

**CI Gate Entry:**
- Location: `extension-kind-gate.mjs` → `main()` (invoked only when run directly)
- Triggers: `node extension-kind-gate.mjs --package-root .` in `.github/workflows/ci.yml`
- Responsibilities: Reads `package.json` kind, dispatches to agent or workflow validator, exits 0/1

## Architectural Constraints

- **No runtime code:** The agent has zero TypeScript/JavaScript source files that execute at agent runtime. The `tsconfig.json` references a `src/` directory but none exists — `tsconfig.json` is a template artifact from repo extraction.
- **LLM model pinning:** `preferredProvider: "openai"`, `preferredModel: "gpt-5.5"` hardcoded in `cinatra/oas.json` (`$referenced_components.review.data.cinatra_llm`)
- **Output envelope required:** The `{"findings":[...]}` wrapper is non-negotiable; a bare JSON array breaks WayFlow's DataFlowEdge key extraction at runtime.
- **Zero external dependencies in gate:** `extension-kind-gate.mjs` uses only Node.js builtins — no npm installs required for CI kind-gate step.
- **Source mirror model:** This repo declares no `@cinatra-ai/*` in `dependencies`/`devDependencies`; monorepo provides and builds host-internal packages. Standalone CI skips install/typecheck/test for source mirrors.
- **Global state:** None. `extension-kind-gate.mjs` is pure (returns values, no module-level mutable singletons).
- **Circular imports:** Not applicable — no multi-module runtime code.

## Anti-Patterns

### Bare JSON array output

**What happens:** LLM returns `[{...}]` instead of `{"findings":[{...}]}`
**Why it's wrong:** WayFlow's `DataFlowEdge` extracts `findings` key by name; a bare array causes `Cannot index array with string "findings"` runtime failure
**Do this instead:** Always return `{"findings":[...]}` — even for parse errors and empty findings (`{"findings":[]}`)

### Duplicating deterministic lint checks in LLM skill

**What happens:** The skill re-checks things the deterministic lint owns (literal secrets, untrusted MCP URLs, `llm-bridge` wiring, package metadata)
**Why it's wrong:** Duplicates authority, creates conflicting signals, and wastes LLM context
**Do this instead:** The SKILL.md's "What you DO NOT check" section enumerates owned domains; keep LLM review to design taste only (`skills/design-review-methodology/SKILL.md` lines 47–54)

## Error Handling

**Strategy:** Fail-safe with structured output. If `oasJson` fails to parse, the agent returns a valid `{"findings":[{...}]}` envelope with a single `unparseable_oas` suggestion rather than crashing.

**Patterns:**
- Parse failure → return `{"findings":[{"code":"unparseable_oas","severity":"suggestion",...}]}`
- No design issues found → return `{"findings":[]}`
- CI gate errors → collected into `string[]` and printed; process exits 1

## Cross-Cutting Concerns

**Logging:** CI gate uses `console.log` / `console.error` for gate pass/fail output. No runtime logging in the agent flow itself (stateless LLM call).
**Validation:** Two layers — deterministic retired-primitive scan in `extension-kind-gate.mjs` (CI-time), LLM design review at runtime via skill.
**Authentication:** Not applicable at the agent level. Platform handles `{{CINATRA_BASE_URL}}` auth.

---

*Architecture analysis: 2026-06-09*
