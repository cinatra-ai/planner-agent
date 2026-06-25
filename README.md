# Planner Agent

An advisory design-review helper for the Cinatra agent-authoring experience. When you create or edit an agent in the /chat interface, the Planner Agent reads the OAS Flow definition under review and returns plain-language suggestions on scope, node choice, input/output naming, sub-agent decomposition, and where a human-in-the-loop checkpoint belongs. It is advisory — it surfaces design-taste improvements that a deterministic linter cannot see.

**Purpose.** The agent receives an `oasJson` string (a serialized OAS Flow body), an optional `packageSlug`, and an optional `reviewContext` hint. It returns a `findings` JSON object listing `ReviewFinding` suggestions with severity `"suggestion"`. A bare JSON array response fails the DataFlowEdge extraction — the agent always wraps output in `{"findings":[...]}`.

**Configuration.** No credentials are needed to trigger a review; the agent is dispatched internally by the Cinatra platform when you author a non-trivial agent in /chat. The `cinatra_llm` block in the OAS selects the underlying model; this is platform-managed.

**Development.** Clone this repo, run `node extension-kind-gate.mjs` to validate the manifest and README, then update `cinatra/oas.json` and `skills/design-review-methodology/SKILL.md` to change review behaviour. Findings not covered here — literal-secret scanning, untrusted MCP URL scanning, package metadata — belong to companion reviewer agents, not this one.

**Troubleshooting.** If the agent returns `{"findings":[{"code":"unparseable_oas",...}]}`, the `oasJson` input was not valid JSON. If findings is always empty, confirm the OAS has at least one `ApiNode` or `AgentNode` in `$referenced_components`.

## Works with

- Cinatra /chat agent-authoring experience (dispatched automatically via agent_source_review)

## Capabilities

- Review a draft OAS Flow agent definition for scope clarity and node choice
- Suggest cleaner input and output naming and flag over-specified interfaces
- Recommend whether work should split into sub-agents or stay a single node
- Flag missing or misplaced human-in-the-loop checkpoints relative to LLM steps
- Surface design-taste improvements the deterministic linter cannot catch
