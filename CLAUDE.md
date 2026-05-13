# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
pnpm install                     # Node 20+ (CI pins 20), pnpm 9+ (CI pins 9)
pnpm run validate                # JSON/YAML/manifest/frontmatter structural checks
pnpm test                        # vitest: 50 tests total — validate.test.js (14) + render-capabilities.test.js (8) + 4 hook test files (28)
pnpm run test:watch              # vitest watch mode while iterating
pnpm run build                   # emits dist/tool-definitions.yaml (for S3 publish, not runtime)

# Single-test file
pnpm exec vitest run scripts/validate.test.js
pnpm exec vitest run skills/run-automation-suite/scripts/render-capabilities.test.js
```

CI (`.github/workflows/ci.yml`) runs `pnpm run validate && pnpm test` on push/PR to `main` with Node 20 / pnpm 9. Both must pass before a PR can merge. There is no lint step.

## Fork sync (this working copy only)

This checkout is a fork — `origin` points at `jeremylongshore/automate`, `upstream` at `kobiton/automate`. Keep `main` in sync before cutting a PR branch:

```bash
git fetch upstream
git checkout main && git merge --ff-only upstream/main && git push origin main
```

Feature branches should always be cut from a freshly-synced `main`.

## Architecture

**This is a thin plugin pointing at a remote MCP server.** Nothing in this repo implements the 12 Kobiton tools — they live server-side at `api.kobiton.com/mcp`. The repo is manifests + one skill + reference schemas.

```
┌────────────────┐  OAuth/API-key ┌──────────────────────────┐
│  Claude Code   │ ──────────────▶│  api.kobiton.com/mcp     │
│  (this plugin) │   HTTPS        │  (12 tools live here)    │
└────────┬───────┘                └──────────┬───────────────┘
         │                                   │
         │ runs locally                      │ source of truth mirrored to
         ▼                                   ▼
  skills/run-automation-suite         tools/*.yaml  ──build──▶  dist/tool-definitions.yaml  ──▶  S3
  agents/ (Claude Code only)          (reference schemas only,
  hooks/  (Claude Code only,           NOT consumed at runtime —
          advisory-only)               server is authoritative)
  AGENTS.md (cross-tool brief
          for Gemini CLI, Codex,
          ChatGPT Apps SDK, etc.)
```

**Cross-tool brief at `AGENTS.md`:** the repo root has an `AGENTS.md` file consumed by Gemini CLI (via `contextFileName`), Codex CLI, GitHub Copilot CLI, and ChatGPT Apps SDK as their equivalent of `SKILL.md`. When extending the skill's workflow or known-limitations list, mirror substantive changes into `AGENTS.md` so non-Claude clients stay current.

**Three `.mcp.*.json` variants, one purpose each:**

| File | When loaded | Auth |
|------|-------------|------|
| `.mcp.json` | default, user installs | OAuth 2.1 browser flow |
| `.mcp.apikey-example.json` | CI/headless — user copies over `.mcp.json` | `Authorization: ${KOBITON_AUTH}` (base64 `user:key`) |
| `.mcp.dev-local.json` | Kobiton dev work against `localhost:3000/mcp` | — |

**The one piece of runtime code in this repo:** `skills/run-automation-suite/scripts/render-capabilities.js`. It reads `references/templates/appium.ejs`, applies CLI-flag values + hardcoded defaults, and emits JSON Appium capabilities for the skill to compare against the user's test script. This is the only code that executes on the user's machine at skill-invocation time.

**Tool schemas are reference documents, not runtime.** `tools/devices.yaml`, `sessions.yaml`, `apps.yaml` describe the 12 tools' input schemas so humans and the Kobiton backend stay in sync. `pnpm run build` concatenates them into `dist/tool-definitions.yaml` that Kobiton publishes to S3. Changing a YAML here does **not** change plugin behavior — the MCP server is authoritative.

## When adding a new tool or skill

Three files must stay in sync or CI fails:

1. The YAML (`tools/<domain>.yaml`) or skill dir (`skills/<name>/`)
2. `scripts/validate.js` — add filename to `toolFiles` array or skill dir to `skillDirs` array
3. `scripts/validate.test.js` — update `setupValidProject` fixtures to match

Tool name must be camelCase ≤64 chars. Skill dir is kebab-case; skill file is uppercase `SKILL.md` with frontmatter `name` + `description`.

Tool YAML must include `inputSchema` (JSON Schema). Annotation rules:

| Tool verb | `readOnlyHint` | `destructiveHint` | `idempotentHint` | `openWorldHint` |
|-----------|:--------------:|:------------------:|:----------------:|:----------------:|
| `list*`, `get*` | true | false | true | false |
| `create*`, `upload*`, `reserve*` | false | false | false | false |
| `confirm*` | false | false | true | false |
| `terminate*`, `delete*` | false | true | false | false |

`openWorldHint: false` is uniform across all 12 Kobiton tools (server is bounded, not open internet).

Tool response payloads **must stay under 25,000 tokens** — trim in the backend handler (not the schema).

Skill structure: numbered `### N. Step` blocks, imperative tone written FOR Claude (not the end user), ask on ambiguity / infer from context, always end with an error-handling section and a summary step.

**When adding a new agent** (`agents/<name>.md`):

- Clean Anthropic agent spec only — `name` + `description` + optional `tools` allowlist. Do NOT use deprecated IS-extension fields (`capabilities`, `expertise_level`, `activation_priority`).
- `description` ≤ 200 characters (marketplace warning threshold).
- Body should cite source-of-truth references it reads from (SKILL.md, `references/capabilities.md`, R2 audit findings linked to upstream).

**When adding a new hook** (`hooks/scripts/<name>.mjs`):

- **Advisory-only** — no authenticated API calls from hook scripts. The agent has authenticated MCP tools already; hooks only inject text into context.
- Use `${CLAUDE_PLUGIN_ROOT}` not `${CLAUDE_PROJECT_DIR}` in `hooks.json`.
- Exec form only: `"command": "node", "args": ["${CLAUDE_PLUGIN_ROOT}/hooks/scripts/X.mjs"]`. Not shell form.
- `hookSpecificOutput` envelope for decisions, not top-level `decision` (top-level is silently ignored on PreToolUse).
- Co-locate test file `hooks/scripts/<name>.test.mjs` covering valid input, boundary cases, missing fields, malformed JSON, and PII-leakage negative tests.
- See `hooks/THREAT-MODEL.md` for the 9 threat categories (T1-T9) that any new hook must address.

## Commits & PRs

- **DCO sign-off is required.** Use `git commit -s`. PRs without `Signed-off-by:` will not be merged.
- **Conventional Commits:** `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`.
- **Branches:** `feat/…`, `fix/…`, `docs/…`, `chore/…` (kebab-case description).
- **Style:** YAML 2-space, no trailing whitespace; JS no-semicolons, single-quotes, Stroustrup braces; Markdown one-sentence-per-line where practical.
- **Review:** 1 maintainer approval, 3-business-day initial review SLA, stale after 14 days inactive.
- **Releases:** maintainers only (bump `.claude-plugin/plugin.json` version, add `CHANGELOG.md` entry, tag `vX.Y.Z`, push). Contributors do not bump versions.

## Security

Vulnerabilities in **this plugin** → email `security@kobiton.com` (never a public GitHub issue). Vulnerabilities in the **Kobiton platform** (API, portal, devices) → Kobiton Trust Center.
