# Technology Stack

**Analysis Date:** 2026-06-09

## Languages

**Primary:**
- JSON — Agent definition (`cinatra/oas.json`), package manifest (`package.json`)
- JavaScript (ESM) — CI gate utility (`extension-kind-gate.mjs`)
- Markdown — Skill definition (`skills/design-review-methodology/SKILL.md`)

**Secondary:**
- TypeScript — TypeScript compiler config present (`tsconfig.json`); no `.ts` source files are tracked in this extracted mirror (the monorepo owns those). Config targets ES2023 with ESNext modules and `bundler` module resolution.

## Runtime

**Environment:**
- Node.js 24 (pinned in `.github/workflows/ci.yml` via `actions/setup-node@v4`)

**Package Manager:**
- pnpm (via corepack — `corepack enable` is the first CI step)
- Lockfile: not committed (CI runs `pnpm install --no-frozen-lockfile` for standalone repos; this repo is a source mirror so standalone install is skipped)

## Frameworks

**Core:**
- Cinatra OAS Flow 26.1.0 — the agent runtime framework; the entire agent is declared as a JSON OAS document at `cinatra/oas.json`

**Testing:**
- Not applicable — this is a source mirror. No test runner config detected. Tests are owned and run by the cinatra monorepo.

**Build/Dev:**
- `extension-kind-gate.mjs` — self-contained, zero-dependency Node.js script for CI OAS validation; uses only Node built-ins (`fs`, `path`)
- TypeScript compiler (`tsc`) — referenced in CI typecheck step; no local `typescript` devDependency (CI falls back to `npx -y -p typescript tsc --noEmit` for standalone repos)
- `corepack` — pnpm version management

## Key Dependencies

**Critical:**
- `@cinatra-ai/planner-agent` (this package, v0.1.0) — the agent itself; no runtime npm dependencies declared
- Cinatra LLM Bridge (`{{CINATRA_BASE_URL}}/api/llm-bridge`) — the sole execution backend called by the agent's `review` ApiNode; declared inline in `cinatra/oas.json`

**Infrastructure:**
- `cinatra-ai/.github` reusable workflow — release pipeline (`reusable-extension-release.yml@main`) used by `.github/workflows/release.yml`
- GitHub Actions — CI/CD platform; workflows in `.github/workflows/`

## Configuration

**Environment:**
- `CINATRA_BASE_URL` — runtime template variable injected by the Cinatra platform into the OAS `review` node URL (`{{CINATRA_BASE_URL}}/api/llm-bridge`); not an `.env` file variable
- `.npmrc` — present (existence noted; contents not read)
- No `.env` files detected

**Build:**
- `tsconfig.json` — standalone TypeScript config; `outDir: dist`, `rootDir: src`, strict mode, `verbatimModuleSyntax`
- `package.json` — declares `cinatra.apiVersion: cinatra.ai/v1`, `cinatra.kind: agent`, `cinatra.dependencies: []`

## Platform Requirements

**Development:**
- Node.js 24+
- pnpm via corepack
- Access to cinatra monorepo workspace for any TypeScript compilation or test execution (this repo is a source mirror)

**Production:**
- Deployed and executed by the Cinatra platform as a Flow 26.1.0 agent node
- Published to `registry.cinatra.ai` via the Cinatra Marketplace submission pipeline (not direct npm/Verdaccio publish)
- LLM inference provided by the platform's `/api/llm-bridge` endpoint

---

*Stack analysis: 2026-06-09*
