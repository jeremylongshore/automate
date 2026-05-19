# Consolidated server-side recommendations (M1-M3 engagement)

> **Status:** captured 2026-05-19 at engagement close, consolidating server-side recommendations from R1 + R2 + R3 reviews of the Kobiton MCP plugin. Each finding below was filed during the engagement as a `kobiton/automate` audit issue (now closed); this file is the single navigable reference for the Kobiton product / platform engineering team.
>
> **Why a `docs/` artifact and not a code change:** the recommendations below are server-side asks (`api.kobiton.com/mcp` and the broader Kobiton backend). The plugin repo cannot ship code that changes server behavior. This file lives in the plugin repo so anyone reading the plugin source has one place to scan the engagement output — without needing to hunt across closed GitHub issues, R-series PDFs, or partner deliverables.
>
> **Plugin-side mitigations** (where they exist) are noted per finding and shipped via parallel close-out PRs (`#67` agents, `#70` hooks, PR-C `listSessions` default, PR-B post-R3 tool doc-drift descriptions).
>
> **Where to find longer narrative:** R2 deliverable `000-docs/021-AA-AACR-r2-mid-cycle-deliverable-report.md` (campaign findings F11-F35), R3 deliverable `000-docs/049-AA-AACR-r3-final-review-deliverable-report.md` (spec-conformance F36-F44), R3 § 6 + `docs/post-r3-tool-doc-drift.md` (F45-F50). Raw fixture payloads for empirically-probed findings are on file with Intent Solutions and available on request.

## Table of contents

| Cluster | Findings | Closed audit issues |
|---|---|---|
| [§ 1 Read-side shape consistency](#1-read-side-shape-consistency) | F14, F16, F17, F45-F50 | `#35`, `#55-#60` |
| [§ 2 Async-state visibility](#2-async-state-visibility) | F25, F26, F32, F33, F40 | `#34`, `#36`, `#40` |
| [§ 3 Error and conflict semantics](#3-error-and-conflict-semantics) | F22, F23, F24 | `#33` |
| [§ 4 Per-command session data](#4-per-command-session-data) | F18, F19, F20 | `#37` |
| [§ 5 MCP server primitives](#5-mcp-server-primitives) | F36, F37, F38, F42 | `#39` |
| [§ 6 OAuth advertisement](#6-oauth-advertisement) | F41a-e | `#41` |
| [§ 7 Above-spec — server-side observability](#7-above-spec--server-side-observability) | F44 | `#42` |
| [§ 8 Lower-priority observations](#8-lower-priority-observations) | F11-F13, F15, F21, F23, F24, F27, F28, F29-F31, F34, F35 | (no separate filings) |

---

## 1. Read-side shape consistency

Findings in this cluster cover divergences between what the plugin's tool descriptions advertise and what the server actually returns, plus inter-endpoint disagreements. The plugin-side description amendments shipped in [PR-B](https://github.com/kobiton/automate) narrow what consumers see today; the server-side fixes below would resolve the underlying drift.

### F14 — `listSessions` 25k-token cap overflow

**Closed audit issue:** [`kobiton/automate#35`](https://github.com/kobiton/automate/issues/35) (cluster with F16 + F17).

**Current observation:** Claude Code's MCP client applies a 25k-token cap on tool responses. `listSessions(limit=20)` responses with verbose `execution_data` per session can approach or exceed this cap; the R2 audit empirically observed responses near the cap silently dropping fields or sessions at the client. Plugin-side mitigation via PR-C lowers the default `limit` to 10 with headroom; this does not solve the underlying overflow shape.

**Recommended server-side change:** add explicit pagination metadata (`next_page_token` / `has_more`) and an opt-in `response_compact: true` flag that drops the verbose `execution_data` for list views. Pair with F40 (`notifications/list_changed`) so consumers can subscribe to delta updates instead of polling pages.

### F16 — Field-naming divergence between `getSession` and `getSessionArtifacts`

**Closed audit issue:** [`kobiton/automate#35`](https://github.com/kobiton/automate/issues/35).

**Current observation:** `getSession` and `getSessionArtifacts` use divergent field shapes for the same data (e.g., `video_url` vs `video.url`-style nesting). Consumers reading both endpoints must reconcile the shapes case-by-case.

**Recommended server-side change:** adopt a single canonical field shape across both endpoints. The R2 audit suggested nested form (`video.url`, `log.url`) over flat snake_case for forward extensibility, but the priority is consistency over either convention.

### F17 — xium URL asymmetry between `getSession` and `getSessionArtifacts`

**Closed audit issue:** [`kobiton/automate#35`](https://github.com/kobiton/automate/issues/35).

**Current observation:** `getSessionArtifacts` exposes the xium URL; `getSession` does not. Consumers who only need session metadata are forced to make a second call (and risk hitting the 25k-token overflow per F14).

**Recommended server-side change:** surface the xium URL in `getSession` as well, or document explicitly that artifact URLs require the artifacts endpoint.

### F45-F50 — Post-R3 tool-description doc drift

**Closed audit issues:** [`#55`-`#60`](https://github.com/kobiton/automate/issues?q=is%3Aissue+author%3Ajeremylongshore+is%3Aclosed) (one per finding).

**Current observation:** see [`docs/post-r3-tool-doc-drift.md`](post-r3-tool-doc-drift.md) for the full catalog. Six places where tool descriptions diverge from the empirical server response (`limit` silently ignored, `has_video` absent, `screenshots[]` not surfaced, `getDeviceStatus` field shrinkage, `is_expired` cross-endpoint disagreement, `confirm_upload` v1/v2 contradiction).

**Recommended server-side change:** for each finding, the server-side fix is to make the response match the advertised shape — either add the documented fields back (`has_video`, `screenshots[]`, the missing `getDeviceStatus` fields), or compute cross-endpoint values consistently (`is_expired`, the `confirm_upload` API version). See each finding's section in `docs/post-r3-tool-doc-drift.md`.

---

## 2. Async-state visibility

Findings in this cluster cover async server operations whose state is not visible to consumers through the existing tool surface. Plugin-side advisory handlers shipped via [PR #70](https://github.com/kobiton/automate/pull/70) route consumers around the gaps today; the server-side fixes below would obviate the route-arounds.

### F25 — `confirmAppUpload` async parser race

**Closed audit issue:** [`kobiton/automate#34`](https://github.com/kobiton/automate/issues/34) (cluster with F26).

**Current observation:** `confirmAppUpload` returns HTTP 200 before the async APK/IPA parser completes. Consumers that immediately call `getApp(appId)` may see `state: parsing_pending`; the parser can subsequently fail with `state: FAILURE_PARSING` minutes later. Without a poll, the consumer believes the upload succeeded and downstream session-reserve calls fail mysteriously.

**Plugin-side mitigation in PR #70:** `advise-app-upload-poll` PostToolUse hook injects an advisory recommending the agent poll `getApp(appId)` until state is `READY` or `FAILURE_PARSING`.

**Recommended server-side change:** extend the `confirmAppUpload` response with a `state` field (`OK` / `FAILURE_PARSING` / `parsing_pending`) returned synchronously, or — better — adopt MCP `resources/subscribe` semantics (per F40) so the consumer can subscribe to parser-complete and parse-error notifications.

### F26 — No `FAILURE_PARSING` diagnostic body

**Closed audit issue:** [`kobiton/automate#34`](https://github.com/kobiton/automate/issues/34).

**Current observation:** when the parser fails, `getApp` returns `state: FAILURE_PARSING` with no diagnostic body. Consumers cannot distinguish "APK is corrupt" from "parser timed out" from "platform mismatch" — every failure looks the same.

**Recommended server-side change:** when `state` is `FAILURE_PARSING`, return a structured `parse_error: { category: "corrupt" | "timeout" | "platform_mismatch" | "unknown", message: "...", detail: ... }` object so consumers can route to the right remediation.

### F32 — W3C-strict `/se/log` rejects legacy `driver.getLogs()`

**Closed audit issue:** [`kobiton/automate#36`](https://github.com/kobiton/automate/issues/36) (cluster with F33).

**Current observation:** Kobiton's Appium endpoint is W3C-strict; the legacy Selenium `POST /se/log` endpoint that `driver.getLogs('logcat')` calls silently fails (the test driver does not receive logs but no error surfaces to the agent). This silently breaks test scripts that depend on log collection during execution.

**Recommended server-side change:** either accept the legacy `/se/log` endpoint at the appropriate W3C-compatible URL, or ship a session-time log-streaming helper (e.g., MCP `resources/subscribe` to a per-session log resource). The plugin-side `SKILL.md` Known Limitations section (PR #65) documents the gap for consumers; that is a documentation workaround, not a fix.

### F33 — Device cleanup cooldown invisible to `getDeviceStatus`

**Closed audit issue:** [`kobiton/automate#36`](https://github.com/kobiton/automate/issues/36).

**Current observation:** after `deleteSession` / session termination, a device enters ~5-min cleanup cooldown during which `reserveDevice` returns `device_unavailable`. The cooldown state is not surfaced by `getDeviceStatus` or `listDevices`, so a consumer that just ended a session and wants to start another sees an opaque conflict.

**Plugin-side mitigation in PR #70:** `advise-pre-terminate-cooldown` and `advise-post-terminate-cooldown` hooks inject advisory notes around `terminateSession` calls.

**Recommended server-side change:** add `cleanup_state: "ready" | "cooling_down"` and `cleanup_until: <ISO8601>` fields to the `getDeviceStatus` and `listDevices` responses for devices in cleanup. Or — preferred — pair with F40 resource subscriptions so the consumer can subscribe to a per-device state stream.

### F40 — Resource subscriptions for async state (spec-layer fold)

**Closed audit issue:** [`kobiton/automate#40`](https://github.com/kobiton/automate/issues/40) (folds F25 + F26 + F33 at the MCP spec layer).

**Current observation:** F25, F26, and F33 are all instances of the same architectural problem: server-side async state changes that consumers cannot observe without polling. MCP `resources/subscribe` is the spec-aligned long-term fix.

**Recommended server-side change:** adopt MCP `resources/subscribe` semantics. Define resources for:

- `kobiton://app/{appId}` — fires on parser state change (F25, F26)
- `kobiton://device/{deviceId}` — fires on availability state change (F33, plus reservation conflicts per F22)
- `kobiton://session/{sessionId}` — fires on state transitions (`START` → `COMPLETE` → `PASSED`/`FAILED`)

Plugin-side advisory hooks in PR #70 are the polling-route-around stopgap; subscriptions would obviate them.

---

## 3. Error and conflict semantics

### F22 — `reserveDevice.conflict_reason` ambiguity

**Closed audit issue:** [`kobiton/automate#33`](https://github.com/kobiton/automate/issues/33).

**Current observation:** when `reserveDevice` returns `device_unavailable`, the error message lumps four distinct failure modes into one ambiguous string:

1. Device is offline (server-side fault — no remediation path for the consumer)
2. Another user is utilizing the device (transient — retry with delay)
3. Device is reserved by someone else (transient — retry, or pick a different device)
4. Public-pool device is fully booked (steady-state — broaden the filter or wait)

Each mode demands a different remediation. The consumer (or agent) cannot reason about which path to take.

**Recommended server-side change:** distinguish the four modes in the response — `conflict_reason: "offline" | "utilizing" | "reserved_by_other" | "pool_unavailable"`. Each enables a different remediation. Optional: include `retry_after_seconds` for transient modes.

### F23 — Idempotency contradiction

**Current observation:** `confirmAppUpload` is documented as idempotent (safe to call multiple times with same `app_path`), but the second call after a successful first call returns a different response shape (lookup-style) than the first (create-style). Consumers that retry on transient network error may build the wrong response-shape assumption.

**Recommended server-side change:** unify the response shape on second-and-subsequent calls — return the same envelope as the first call, just with a `created: false` indicator if the operation was a lookup rather than a creation.

### F24 — Duplicated error structure

**Current observation:** error responses across endpoints carry duplicated wrapper objects (`{ error: { error: { code: ..., message: ... } } }` in some cases). Consumers cannot rely on a single canonical path to the error code.

**Recommended server-side change:** flatten to one canonical shape — `{ error: { code, message, detail } }`. Apply uniformly across all 12 tool endpoints.

---

## 4. Per-command session data

**Closed audit issue:** [`kobiton/automate#37`](https://github.com/kobiton/automate/issues/37) (F18 + F19 + F20 cluster).

### F18 — Per-command session data not exposed

**Current observation:** `getSession` returns `execution_data.all_command_data_available: true` but no plugin tool exposes the per-command list. Consumers that want to inspect what happened in a session must use the Kobiton portal manually. This directly blocks the "drive a session, capture it as a reusable test asset" use case that motivates the broader `saveTestRun` story.

**Recommended server-side change:** add `getSessionCommands(sessionId)` returning per-step `[{ commandIndex, command, target, action, screenshotUrl, ts }]`. Ship this independently of any test-asset-capture endpoint; it is a plugin DX win on its own.

### F19 — Per-command screenshot URLs not exposed

**Current observation:** the Kobiton portal shows per-command screenshots; the plugin surface does not. Same root cause as F18.

**Recommended server-side change:** include `screenshotUrl` in the per-step records returned by `getSessionCommands` (above).

### F20 — Assertion semantics gap

**Current observation:** no plugin tool exposes assertion semantics — neither at the session level (did this session represent a passing test) nor at the per-step level (which step assertions failed). Consumers driving sessions cannot capture the resulting test case with assertion state.

**Recommended server-side change:** make "draft test case with blank assertions" the explicit first-class output of a hypothetical `saveTestRun(sessionId, appId, name)` flow. Assertion population becomes a separate `annotateTestCase(testCaseId, annotations)` flow. The pieces are independently shippable.

---

## 5. MCP server primitives

**Closed audit issue:** [`kobiton/automate#39`](https://github.com/kobiton/automate/issues/39) (F36 + F37 + F38 + F42 cluster).

The R3 spec-conformance audit (against [`code.claude.com/docs/en/mcp`](https://code.claude.com/docs/en/mcp) and the [MCP 2025-06-18 spec](https://modelcontextprotocol.io/specification/2025-06-18)) surfaced four MCP server primitives that `api.kobiton.com/mcp` does not currently declare. Plugin-side adjacent work shipped via [PR #67](https://github.com/kobiton/automate/pull/67) (three specialized agents that fill some of the same shape on the host side) and [PR #70](https://github.com/kobiton/automate/pull/70) (advisory hooks that route around several gaps); the server-side primitives below would be the spec-aligned fix.

### F36 — `instructions` field absent from `initialize`

**Current observation:** the MCP spec's `initialize` response carries an `instructions` field that lets a server tell every connecting client how to use it (authentication setup, common workflows, known limitations). `api.kobiton.com/mcp` does not populate this field; consumers that don't read the plugin's `SKILL.md` (e.g., non-Claude hosts, raw MCP clients) have no in-band orientation.

**Recommended server-side change:** populate the `instructions` field in the `initialize` response with a one-paragraph orientation: what the server does, where to start (`listDevices`, `uploadAppToStore`), where to find authentication setup, where to find known limitations. Pull from the existing `SKILL.md` content.

### F37 — `resources/*` capability absent

**Current observation:** the MCP spec defines `resources/list`, `resources/read`, `resources/subscribe`, `resources/templates/list` as server primitives for exposing addressable state. `api.kobiton.com/mcp` declares no `resources` capability. Consumers cannot subscribe to async state changes (F25, F26, F33, F40 — see § 2) or address Kobiton resources directly.

**Recommended server-side change:** declare `resources` capability in the server's `initialize` response and expose a starting resource set: `kobiton://device/{id}`, `kobiton://session/{id}`, `kobiton://app/{id}`, `kobiton://artifact/{type}/{sessionId}`. `resources/subscribe` for async state changes ties to the F40 recommendation.

### F38 — `prompts/*` capability absent

**Current observation:** the MCP spec defines `prompts/list` and `prompts/get` as server primitives for canonical prompt templates the server wants clients to use. `api.kobiton.com/mcp` declares no `prompts` capability. The plugin's `run-automation-suite` skill is essentially what would be exposed as `prompts/automation_suite_workflow` if the server declared the capability.

**Recommended server-side change:** declare `prompts` capability and expose `prompts/automation_suite_workflow` (mirroring the existing skill), `prompts/triage_failed_session`, `prompts/select_device`. Non-Claude MCP hosts (Cursor, ChatGPT Apps SDK, Continue) consume prompts but not skills — this is the cross-tool equivalent of the skill surface.

### F42 — `notifications/*` and subscriptions (fold)

**Current observation:** the MCP spec defines `notifications/resources/updated`, `notifications/resources/list_changed`, `notifications/tools/list_changed`, `notifications/prompts/list_changed` as server-to-client push primitives. `api.kobiton.com/mcp` does not emit any notifications, so clients cannot observe list mutations or resource state changes without re-polling.

**Recommended server-side change:** once `resources/*` (F37) is declared, emit `notifications/resources/updated` for the relevant addressable resources. `notifications/tools/list_changed` is lower-priority since the tool set is stable; emit on the rare deploys that add/remove tools.

---

## 6. OAuth advertisement

**Closed audit issue:** [`kobiton/automate#41`](https://github.com/kobiton/automate/issues/41).

**Verification posture (important):** the R3 OAuth findings were empirically verified per kobiton/CLAUDE.md ops rule #11. The original R3 Bundle 3 draft claimed `api.kobiton.com/mcp` was missing RFC 9728 (OAuth 2.0 Protected Resource Metadata) and RFC 8414 (OAuth 2.0 Authorization Server Metadata) entirely. The probe (2026-05-12) found that **both are implemented**. What's actually missing is narrower — captured below as F41a-e.

### F41a — `resource_indicators_supported` not declared

**Current observation:** RFC 8707 (Resource Indicators for OAuth 2.0) defines `resource` as an authorization-request parameter. The authorization server's metadata document at `https://api.kobiton.com/.well-known/oauth-authorization-server` does not advertise `resource_indicators_supported: true`, so spec-compliant clients cannot reliably know whether to send the `resource` parameter.

**Recommended server-side change:** add `"resource_indicators_supported": true` to the authorization-server metadata document. (This is a one-field addition to a JSON document; the underlying capability is already implemented.)

### F41b — `WWW-Authenticate` header absent on bad-token 401

**Current observation:** when a bearer token is rejected, the server returns HTTP 401 without a `WWW-Authenticate` response header. RFC 6750 § 3 requires this header on 401 responses so clients can discover the realm and renew the token automatically.

**Recommended server-side change:** emit `WWW-Authenticate: Bearer realm="kobiton-mcp", error="invalid_token", error_description="..."` on every 401 response. This is a single-line response-header addition.

### F41c-e — Lower-priority OAuth-metadata polish

Three smaller items from the same probe:

- **F41c** — protected-resource metadata at `https://api.kobiton.com/.well-known/oauth-protected-resource` does not advertise `resource_documentation` (informational URL where developers can find the resource server's docs).
- **F41d** — token-endpoint metadata could declare `token_endpoint_auth_signing_alg_values_supported` for clients that want to use signed JWT client authentication.
- **F41e** — the authorization-server metadata document does not declare `code_challenge_methods_supported: ["S256"]` even though PKCE S256 is required and supported.

Each is a one-line metadata addition.

---

## 7. Above-spec — server-side observability

**Closed audit issue:** [`kobiton/automate#42`](https://github.com/kobiton/automate/issues/42).

### F44 — Server-side OpenTelemetry instrumentation

**Current observation:** today every observability path lives in the agent host. Claude Code can emit `claude_code.hook` spans from the advisory handlers shipped in PR #70, but those are Claude-Code-specific. Non-Claude MCP clients (Cursor, ChatGPT Apps SDK, Continue, Gemini CLI, Codex CLI) have no equivalent. The result: a Kobiton platform engineer investigating "why are agent-driven sessions failing" must reconstruct the path from N different agent hosts' telemetry surfaces, none of which see the server side.

**Recommended server-side change:** instrument `api.kobiton.com/mcp` with OpenTelemetry — vendor-neutral, exportable to any backend the operator chooses. The signal taxonomy:

- **Spans:** `mcp.tool_call` (per call) with `tool.name`, `tool.user_intent`, `tool.duration_ms`, `tool.outcome`; `mcp.session_lifecycle` (per session-bearing tool chain); `mcp.async_state_change` (parser-complete, cleanup-cooldown-clear).
- **Log events:** structured access-log-shape events for every tool call with `user_intent` populated.

Cross-references: this would close the F44 ask uniformly across all MCP clients — the plugin-side advisory hooks in PR #70 fill the same gap on Claude Code only, narrowly. Gating env vars (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME=kobiton-mcp`) make this a deploy-time toggle with no plugin-side coupling.

Detailed signal taxonomy + gating var discussion: R3 deliverable `000-docs/049-AA-AACR` § 5.

---

## 8. Lower-priority observations

These were filed in R2 deliverable § 4 + § 10 as completeness-class observations; none were separately filed as standalone upstream audit issues because they did not block R2 → R3 progression. Captured here for completeness.

### F11-F13, F15 — R2 minor doc-shape observations

Small documentation-surface observations from R2 (formatting consistency, missing `Use when` phrase in one skill section, etc.). Largely addressed in R1's documentation-shape fix campaign and the close-out PRs #64 + #65 + #66. No further server-side change needed.

### F21 — Inconsistent timestamp formats across endpoints

**Current observation:** session and app timestamps mix `ISO 8601` and Unix-epoch formats across different fields and endpoints. Consumers parsing timestamps must handle both forms.

**Recommended server-side change:** standardize on ISO 8601 with `Z` suffix (`2026-05-19T12:34:56.789Z`) for every timestamp field across every endpoint.

### F27 — Cache headers absent

**Current observation:** read-side endpoints (`listDevices`, `listApps`, `getApp`) do not emit `Cache-Control` / `ETag` / `Last-Modified` headers. Consumers cannot reason about freshness or implement client-side caching.

**Recommended server-side change:** emit `Cache-Control: max-age=N` appropriate to the resource type, plus `ETag` for `getApp` and `getDeviceStatus`. Hot list endpoints (`listSessions`) probably stay no-cache; reference-data endpoints (`listDevices` for a stable fleet) can carry meaningful caching headers.

### F28 — Rate-limit headers absent

**Current observation:** the server presumably has request-rate limits but does not emit `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` headers. Consumers cannot back off proactively; they discover rate limits only via 429 responses.

**Recommended server-side change:** emit standard rate-limit headers on every response. Pair with `Retry-After` on 429 responses.

### F29-F31 — Artifact-triage observations

R2's exp-05 (artifact-triage) experiment surfaced three observations about session-failure debuggability:

- **F29** — crash logs are surfaced via `getSessionArtifacts.crashLogs[]` only when crashes occurred; "no crashes" returns an empty array (good shape — no change recommended).
- **F30** — session video URLs have a finite TTL; consumers caching URLs across hours risk hitting expired links.
- **F31** — log files include partner-system internal IDs that consumers cannot resolve back to anything actionable.

**Recommended server-side change (F30):** document video URL TTL in the `getSessionArtifacts` response (`video.expires_at: <ISO8601>`). **F31** is lower priority — could be folded into `instructions` (F36) as part of the in-band orientation.

### F34, F35 — Cross-device parity cluster observations

R2's exp-03 (cross-device parity matrix, 5 fresh sessions across 5 devices) surfaced two cluster-shape observations:

- **F34** — `getSession.device_name` returns the human-readable device name; `listSessions[i].device_name` returns the same for some sessions and the UDID for others. Inconsistent.
- **F35** — boot-time-to-session-ready varies 18s-90s across the 5-device parity matrix with no exposed signal as to why. Not a finding requiring a fix, just an operational observation that consumers may want surfaced (e.g., a `device.warm_boot_estimate_seconds` field).

**Recommended server-side change (F34):** align field semantics across `getSession` and `listSessions`. **F35** is informational — could fold into a future device-metadata enrichment.

---

## How to read this catalog

Each finding above maps to one or more closed `kobiton/automate` audit issues (where filed) and to a section of the R-series deliverables (`000-docs/021-AA-AACR-r2-...md`, `000-docs/049-AA-AACR-r3-...md`). The recommendations are written for the Kobiton product / platform engineering team to scan, prioritize, and route to the right server-side owner — they are not stack-ranked here.

**For prioritization help**, see R2 deliverable § 4 (top 5 R3-window plugin-DX findings, ranked) and R3 deliverable § 3.4 (Track 7 close-out matrix). The plugin-side mitigations and advisory hooks shipped via the close-out PRs (`#67` agents, `#70` hooks, PR-B descriptions, PR-C `listSessions` default) route consumers around the highest-impact gaps today; the server-side changes above would make those route-arounds obsolete.

Refs `kobiton/automate#33, #34, #35, #36, #37, #39, #40, #41, #42, #55, #56, #57, #58, #59, #60` (all closed).

- Jeremy Longshore
intentsolutions.io
