# External Integrations

**Analysis Date:** 2026-06-09

## APIs & External Services

**LLM Bridge (Cinatra Platform):**
- Cinatra `/api/llm-bridge` — the sole external call made by this agent; performs the advisory design review via an LLM
  - SDK/Client: HTTP POST via Cinatra `ApiNode` (OAS declaration in `cinatra/oas.json`, node id `review`)
  - Auth: Handled by the Cinatra platform at runtime; no API key is declared in the OAS
  - Preferred LLM: OpenAI `gpt-5.5` (`cinatra_llm.preferredProvider: "openai"`, `cinatra_llm.preferredModel: "gpt-5.5"` in `cinatra/oas.json`)
  - Template variable: `{{CINATRA_BASE_URL}}` injected by the Cinatra runtime

**Cinatra Marketplace / Registry:**
- `registry.cinatra.ai` — publish destination for released versions
  - Not a direct npm/Verdaccio publish; submission goes through the marketplace MCP proxy (`extension-submit-for-review` saga)
  - Orchestrated by the reusable GitHub Actions workflow `cinatra-ai/.github/.github/workflows/reusable-extension-release.yml@main`
  - Auth: `CINATRA_MARKETPLACE_VENDOR_TOKEN` org secret (GitHub Actions secret, not present in this repo directly)

## Data Storage

**Databases:**
- Not applicable — this agent is stateless; it receives an OAS JSON body as input and emits design findings. No database is read or written.

**File Storage:**
- Not applicable — no file storage integration.

**Caching:**
- Not applicable — no caching layer.

## Authentication & Identity

**Auth Provider:**
- Cinatra Platform (runtime-managed) — the `ApiNode` call to `/api/llm-bridge` carries `agent_id: "planner-agent"` and `agent_run_id` as part of the POST body. Platform-level auth is handled by the runtime; no OAuth/JWT/API key is declared in the OAS or package.
- GitHub OIDC — used in the release workflow (`id-token: write` permission) for build-provenance attestation via `attestations: write`

## Monitoring & Observability

**Error Tracking:**
- Not detected — no third-party error tracking SDK integrated.

**Logs:**
- The agent emits `{"findings":[...]}` JSON as its only output. Logging/tracing is handled by the Cinatra platform runtime, not by this package directly.

## CI/CD & Deployment

**Hosting:**
- Cinatra Platform — agent is executed as a Flow 26.1.0 node within the Cinatra runtime environment.

**CI Pipeline:**
- GitHub Actions (`.github/workflows/ci.yml`) — runs on push/PR to `main`:
  1. Classifies repo as source mirror vs standalone
  2. Skips standalone install/typecheck/test (source mirror — monorepo owns those)
  3. Runs `npm pack --dry-run` for package shape validation
  4. Runs `node extension-kind-gate.mjs --package-root .` (agent OAS validation gate)
- GitHub Actions (`.github/workflows/release.yml`) — triggered on GitHub Release publish; delegates entirely to `cinatra-ai/.github` reusable workflow

## Environment Configuration

**Required env vars (runtime):**
- `CINATRA_BASE_URL` — Cinatra platform base URL; injected as a template variable into `cinatra/oas.json` at runtime (not a build-time env var)

**Required secrets (CI/CD):**
- `CINATRA_MARKETPLACE_VENDOR_TOKEN` — org-level GitHub Actions secret; consumed by the reusable release workflow; not declared or used in this repo's own files

**Secrets location:**
- GitHub Actions org secrets (managed in `cinatra-ai` GitHub org)
- `.npmrc` file present — contents not read; likely contains registry auth configuration

## Webhooks & Callbacks

**Incoming:**
- Not applicable — the agent is invoked by the Cinatra platform as a Flow node, not via webhook.

**Outgoing:**
- Not applicable — the agent makes one outbound HTTP call (to `/api/llm-bridge`) via the Cinatra platform's `ApiNode` mechanism; no external webhooks are registered.

## Skill Integration

**design-review-methodology skill:**
- Location: `skills/design-review-methodology/SKILL.md`
- Integration: Delivered to the LLM as a system-level skill during execution. The skill is matched when `agent_id` equals `@cinatra-ai/planner-agent`. It provides the design-review methodology, check categories, and output contract instructions directly to the LLM prompt.
- No MCP primitives called by this skill — pure advisory, no fetch/save/mutation.

---

*Integration audit: 2026-06-09*
