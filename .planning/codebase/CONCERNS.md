# Codebase Concerns

**Analysis Date:** 2026-06-09

## Tech Debt

**Hardcoded LLM model reference:**
- Issue: `cinatra/oas.json` hardcodes `"preferredModel": "gpt-5.5"` in the ApiNode data payload. If this model is deprecated or renamed, the agent silently uses a fallback or fails without a code change.
- Files: `cinatra/oas.json` (line 192)
- Impact: Model drift — the agent may use a different model than intended without any warning surfaced to operators.
- Fix approach: Centralise model preference in a versioned config or accept it as an input so it can be overridden without touching the OAS artefact.

**Inline `reviewContext` default is an opaque `"{}"`:**
- Issue: The `reviewContext` input defaults to the string `"{}"` rather than an empty object or typed sub-fields. Callers must serialize/deserialize a JSON blob manually and there is no schema enforcement.
- Files: `cinatra/oas.json` (lines 23-26, 175-177), `skills/design-review-methodology/SKILL.md` (line 13)
- Impact: Any malformed `reviewContext` JSON silently degrades the LLM's context without an error path; callers have no schema contract.
- Fix approach: Either flatten `reviewContext` into typed scalar inputs (`reviewPhase`, `invokedBy`) or add a JSON-schema validation step before the LLM call.

**`extension-kind-gate.mjs` duplicates monorepo gate logic:**
- Issue: The file documents that it ports rules "verbatim" from `scripts/audit/oas-banned-primitives-gate.mjs` in the monorepo (line 8). The `BANNED_PRIMITIVES` and `BANNED_TYPEHINTS` lists are manually mirrored; divergence is a silent risk.
- Files: `extension-kind-gate.mjs` (lines 65-74)
- Impact: If a new banned primitive is added to the monorepo gate but not mirrored here, the extracted-repo CI passes on a violation the monorepo would catch.
- Fix approach: Generate or checksum-verify this list from the monorepo source of truth during the extraction step, rather than maintaining a manual copy.

**`release.yml` is marked dormant:**
- Issue: The release workflow comment states it is "Dormant until the org infra exists (the cinatra-ai/.github reusable workflow + the CINATRA_MARKETPLACE_VENDOR_TOKEN org secret)." Any GitHub Release published before that infra exists will silently fail.
- Files: `.github/workflows/release.yml` (lines 10-12)
- Impact: A published release could create a false sense of "shipped" while the actual marketplace submission job exits with a missing-secret or missing-workflow error.
- Fix approach: Add a pre-check step that asserts the org secret and reusable workflow are available, or gate the job with an explicit environment that does not yet exist so it is visibly blocked.

## Known Bugs

**Bare-array LLM response causes DataFlowEdge failure:**
- Symptoms: If the LLM returns a bare JSON array (`[{...}]`) instead of the required `{"findings":[...]}` envelope, the DataFlowEdge that extracts the `findings` key throws `Cannot index array with string "findings"` and the flow fails.
- Files: `cinatra/oas.json` (line 185), `skills/design-review-methodology/SKILL.md` (lines 38, 62)
- Trigger: LLM model generation drift — the model may omit the wrapping object despite the instruction. This is mentioned explicitly in the skill as a known failure mode, yet no defensive extraction or retry step exists in the OAS.
- Workaround: The system prompt and SKILL.md repeat the instruction multiple times, but this is not a structural guarantee.

## Security Considerations

**LLM prompt injection via `oasJson` input:**
- Risk: The `oasJson` input is injected verbatim into the LLM user prompt (line 186 of `cinatra/oas.json`). A malicious OAS body could contain adversarial instructions designed to manipulate the planner-agent's output.
- Files: `cinatra/oas.json` (lines 186-188)
- Current mitigation: None structural — the system prompt instructs the model to only emit `{"findings":[...]}`, which constrains output format but does not prevent prompt injection from influencing finding content.
- Recommendations: Sanitize or bracket the `oasJson` input before interpolation; consider using a separate API message role for untrusted content if the LLM bridge supports it.

**`.npmrc` file present:**
- `.npmrc` exists at the repo root. Contents not read. May contain a registry auth token for `registry.cinatra.ai`.
- Risk: If `.npmrc` contains an auth token and is accidentally included in a published npm pack, the token is exposed.
- Current mitigation: Unknown — verify `.npmrc` is listed in `.npmignore` or `package.json` `files` allowlist exclusion.

## Performance Bottlenecks

**No streaming or partial-result path:**
- Problem: The agent issues a single blocking POST to `/api/llm-bridge` and waits for a full `findings` JSON response. For large OAS bodies, this is a long cold synchronous wait with no partial delivery.
- Files: `cinatra/oas.json` (lines 182-227)
- Cause: The OAS Flow uses a single ApiNode with no timeout, retry, or streaming config.
- Improvement path: Add a timeout parameter to the ApiNode or split large OAS reviews into chunked sub-agent calls via FlowNode decomposition.

## Fragile Areas

**Single-node OAS flow with no error branch:**
- Files: `cinatra/oas.json` (lines 55-76)
- Why fragile: The control flow is strictly linear: `start → review → end`. There is no error/fallback edge. If the ApiNode call to `/api/llm-bridge` fails (network error, 5xx, timeout), the flow fails opaquely with no structured error output.
- Safe modification: Add a fallback ControlFlowEdge from the `review` node to an error EndNode that emits `{"findings":[{"code":"review_failed","severity":"suggestion","message":"Design review unavailable.","source":"agent-planner"}]}`.
- Test coverage: No tests exist in this repo for the OAS flow behavior. The `extension-kind-gate.mjs` only validates the OAS parses and contains no retired primitives — it does not test runtime behavior.

**`walkLlmStrings` only checks a fixed set of field names:**
- Files: `extension-kind-gate.mjs` (lines 63, 92-105)
- Why fragile: `LLM_VISIBLE_FIELDS` is hardcoded to `["system", "user", "description"]`. If the OAS spec adds new LLM-visible fields (e.g. `assistant`, `context`), retired primitives in those fields will not be caught by the gate.
- Safe modification: Update `LLM_VISIBLE_FIELDS` in sync with any OAS spec field additions.

## Scaling Limits

**LLM context window for large OAS inputs:**
- Current capacity: No input size validation or truncation on `oasJson`.
- Limit: Very large agent OAS bodies (multi-thousand-line flows) may exceed the LLM context window of `gpt-5.5`, producing a truncated or refused response.
- Scaling path: Add a pre-check step that measures `oasJson` token count and either rejects oversized inputs with a structured finding or splits the review into windowed chunks.

## Dependencies at Risk

**`preferredProvider: "openai"` with no fallback:**
- Risk: The OAS hardcodes OpenAI as the preferred provider. If OpenAI is unavailable or the `gpt-5.5` model is deprecated, there is no declared fallback provider.
- Impact: The entire agent becomes unavailable during OpenAI outages.
- Migration plan: Declare a secondary `fallbackProvider` if the `cinatra_llm` schema supports it, or abstract the model/provider selection to a runtime config.

## Missing Critical Features

**No test suite:**
- Problem: The repo ships no tests. The `extension-kind-gate.mjs` contains exported pure functions (`validateAgent`, `validateBpmnSanity`, `validateWorkflowPackageShape`, etc.) that are directly unit-testable but have no tests.
- Blocks: Cannot verify gate correctness after modifying the `BANNED_PRIMITIVES` list or adding new validation rules without running the gate manually against fixture files.

**No input validation before LLM call:**
- Problem: `oasJson` is passed directly to the LLM without validating it is well-formed JSON. The SKILL.md instructs the LLM to handle parse failure gracefully (step 1), but the OAS flow itself has no pre-validation node.
- Blocks: A non-JSON `oasJson` (e.g. plain text, HTML error page) will be sent to the LLM and consume tokens before returning a `unparseable_oas` finding that could have been caught cheaply.

## Test Coverage Gaps

**`extension-kind-gate.mjs` exported functions — zero test coverage:**
- What's not tested: `validateAgent`, `validateWorkflow`, `validateWorkflowPackageShape`, `validateBpmnSanity`, `findWorkflowSidecars`, `parseArgs`, `runGate`.
- Files: `extension-kind-gate.mjs`
- Risk: Regressions in banned-primitive detection or BPMN sanity checks go undetected until a bad OAS or BPMN reaches CI of a downstream repo.
- Priority: High — this is the only CI gate for all extracted agent/workflow repos.

**OAS flow runtime behavior — zero test coverage:**
- What's not tested: DataFlowEdge extraction of `findings`, ApiNode timeout/error behavior, bare-array LLM response handling.
- Files: `cinatra/oas.json`
- Risk: The known bare-array bug (see Known Bugs) cannot be caught by any automated test in this repo.
- Priority: Medium — covered partially by integration tests in the cinatra monorepo, but not verifiable from this standalone repo.

---

*Concerns audit: 2026-06-09*
