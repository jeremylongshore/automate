---
name: run-automation-suite
description: >-
  Run local Appium test scripts against Kobiton devices — guides you through
  app upload, device selection, capability parsing, and local execution.
  Use when the user asks to run mobile tests on Kobiton, validate an APK or
  IPA on real devices, or kick off a Kobiton automation run from a local
  script directory. Trigger with "run kobiton tests" or "execute on kobiton
  devices".
version: 1.0.2
author: Kobiton Inc.
license: MIT
tags: [mobile, testing, appium, automation, devices, kobiton]
compatibility: "Designed for Claude Code; requires Node.js >= 20 and Appium 2.x for local script execution."
allowed-tools: "Read, Write, Edit, Bash(node:*), Bash(python:*), Bash(dotnet:*), Bash(mvn:*), Bash(java:*), Bash(open:*), Bash(xdg-open:*)"
---

# Run Automation Suite

## Overview

Execute Appium-based mobile test automation suites on Kobiton's device cloud. Given a directory of local test scripts, identify the target app, select an available device, parse and reconcile capabilities, run the suite, and surface results back to the user with session links and artifacts.

Use this skill when the user asks to run mobile tests on Kobiton, validate an APK or IPA on real devices, or trigger a Kobiton-hosted automation run from a local script directory.

## Prerequisites

Before beginning the workflow, confirm:

- A Kobiton MCP server connection is configured (one of `.mcp.json`, `.mcp.apikey-example.json`, or `.mcp.dev-local.json`).
- The user has a directory of Appium test scripts (`.js`, `.py`, `.cs` / `.csproj`, or `.java`) ready to execute.
- The render-capabilities helper at `skills/run-automation-suite/scripts/render-capabilities.js` is reachable from the working directory.

## Instructions

### 1. Identify the app

**IMPORTANT: Always ask the user this question, even if they already provided an app file path. Do NOT skip ahead or start uploading automatically.**

Ask the user:

> "Would you like to:"
> 1. Upload a new app build
> 2. Use an existing app from Kobiton Store or a provided URL

Wait for their response before proceeding. Do not call any upload or app-related tools until the user responds.

**If uploading a new app:** Look for .apk, .ipa, .zip files in the project context, or ask the user for the file path. Upload via `uploadAppToStore` (permanent, visible in app repository). This is a three-step process: call the tool to get a pre-signed URL, upload the file via PUT, then confirm the upload.

**If reusing an existing app:** Check `appium:app` field of capabilities in the test script. Call `listApps` with that app version as keywork to check uploaded or not. Let the user pick the version to use (e.g., `kobiton-store:v72107`) if needed

### 2. Select a device

Ask the user which device or platform to target.

Call `listDevices` with the relevant platform filter to show available options.

If the user has a specific device in mind, confirm its availability with `getDeviceStatus`.

Reserve the device with `reserveDevice` if needed.

### 3. Identify script & parse capabilities

Ask the user for the path to their local Appium test script.

Detect the language and runtime from the file extension:

| Extension | Runtime | Command |
|-----------|---------|---------|
| `.js` | Node.js | `node <script> <udid>` |
| `.py` | Python | `python <script> <udid>` |
| `.cs` / `.csproj` | .NET | `dotnet test` |
| `.java` | Java | `mvn test` or `java -cp ...` |

Read the script file and extract key capabilities from the source code. The full field list, the Appium-runtime opt-in special case, and the per-field comparison policy live in [`references/capabilities.md`](references/capabilities.md).

Identify how the UDID is passed into the script (CLI argument, environment variable, or hardcoded) so it can be overridden with the selected device.

**Validate capabilities:** After parsing the script, run the render script to generate the correct capabilities for the selected device and app:

```
node skills/run-automation-suite/scripts/render-capabilities.js \
  --platformName <platform> \
  --udid <udid> \
  --deviceName "<deviceName>" \
  --platformVersion <version> \
  --automationName <automationName> \
  --app <app> \
  --testingType app
```

For web testing, replace `--app <app>` with `--browserName <browser> --testingType web`.

Apply the per-field comparison policy from [`references/capabilities.md`](references/capabilities.md) to reconcile the rendered output against the parsed script capabilities (must-match → auto-edit, suggested-default → ask, user-controlled → leave untouched).

### 4. Confirm & execute

Present a summary to the user before running:

```
Language:     Node.js
Script:       /path/to/test.js
Platform:     Android
Device:       Pixel 4 (9B211FFAZ0017F)
App:          kobiton-store:v72107
Session Name: Verify Appium session
Command:      node /path/to/test.js 9B211FFAZ0017F
```

Wait for user confirmation, then execute the command via the Bash tool **in the background** (use `run_in_background: true`).

**Immediately after launching the script**, wait **2 seconds** (to allow the session to initialize on Kobiton), then open the running session in the user's browser.

**Determine the portal URL:** Read `.mcp.json` to get the MCP server URL, then map it to the portal base URL:

| MCP Server | Portal Base URL |
|------------|----------------|
| `api.kobiton.com` | `https://portal.kobiton.com` |
| `api-test-green.kobiton.com` | `https://portal-test.kobiton.com` |

**Build the launch URL:**

```
<portal-base-url>/devices/launch?id=<deviceId>
```

Where `<deviceId>` is the ID of the selected device from Step 2 (returned by `listDevices`, `getDeviceStatus`, or `reserveDevice`).

**Open the link** in the user's default browser:

| Platform | Command |
|----------|---------|
| macOS | `open <url>` |
| Linux | `xdg-open <url>` |

## Output

After the test executes, collect session artifacts and summarize the run for the user.

### Collect session artifacts

After opening the browser, call `listSessions` with `deviceId=<deviceId>` (from Step 2) and `state='START'` to find the session that just triggered. Use the most recent session (first result) as the match.

Call `getSession` with the matched session ID to get detailed results.

Call `getSessionArtifacts` with the session ID to retrieve:

- Video recording URL
- Device logs URL
- Screenshots
- Test reports

### Summarize

Present a summary to the user:

- Pass/fail status
- Session link in Kobiton portal
- Video recording link
- Key error messages (if failed)
- Execution duration

## Error Handling

- `listDevices` returns empty: suggest broadening filters (remove platform/group constraints) or trying again later when devices free up.
- Upload fails or times out: retry the upload. Pre-signed URLs expire after 30 minutes — if expired, call the upload tool again to get a fresh URL.
- Session stuck in a non-terminal state: poll `getSession` with a reasonable timeout. If still running, offer to call `terminateSession` and retry.
- `reserveDevice` fails (device already taken): call `listDevices` again to find another available device.
- Script execution fails: check error output for missing dependencies (e.g. `wd`, `appium`), incorrect UDID, or network issues. Suggest fixes.

## Examples

The following are demonstration scenarios drawn from the workflow's currently-supported branches. Each one maps to a documented Step in the workflow above. Adjust the specific apps, devices, and prompt phrasing to whatever Kobiton's customers most often ask for.

### Run an Android suite on the first available matching device

> "Run the test scripts in `./tests/checkout/` on a Pixel 7 if available, otherwise any Android device with API 33 or higher."

The skill identifies the test directory, queries `listDevices` filtered to Android API 33+ (preferring Pixel 7), reserves the device, parses capabilities from the test config, executes the suite via the Bash tool in the background, opens the live session in the user's browser, and returns the Kobiton portal session URL with collected artifacts (video, logs, screenshots, reports).

### Reuse an existing app build on a specific device + version

> "Use the app at `kobiton-store:v72107` and run my login tests on a Galaxy S22 with Android 13."

The skill takes the existing-app branch in Step 1 (no upload), confirms the requested device is available via `getDeviceStatus`, reserves it via `reserveDevice`, parses the user's login tests, executes, and returns the session summary on completion.

### Run a browser-based web test instead of a native app

> "Run my Selenium tests in `./tests/web/` against Chrome on a recent Android device."

The skill renders capabilities with `--browserName chrome --testingType web` (the `web` branch of Step 3) instead of `--app`, parses the script as a browser-based test, and collects the same artifact set on completion.

## Resources

The links below cover the canonical Kobiton and Appium surfaces relevant to this skill. Kobiton-side, the team should refine specific deep-link URLs (e.g., the Kobiton capability builder's exact docs path) to whatever the platform team currently considers authoritative.

- [Kobiton platform overview](https://kobiton.com) — the device cloud the skill targets; account signup
- [Kobiton documentation](https://docs.kobiton.com) — Kobiton-side reference for desired capabilities, vendor extensions, and platform behavior
- [Appium official documentation](https://appium.io) — Appium project reference, including driver-specific capabilities and the Appium 2.x driver model
- [Plugin source on GitHub](https://github.com/kobiton/automate) — issues, contributions, releases
- [Sample prompt examples](https://github.com/kobiton/automate/blob/main/docs/examples.md) — one natural-language prompt example per tool, maintained alongside the plugin

### Related skills

<!-- Cross-link other Kobiton-published skills here as they ship. -->

## Known Limitations

Active gaps documented in the R2 audit (2026-05-04, Intent Solutions pilot). Listed here so the agent can plan around them — most require server-side changes at `api.kobiton.com/mcp` and are tracked at the linked upstream issues. Plugin-side mitigations noted where they exist.

- **`confirmAppUpload` returns before the async parser finishes** ([upstream #34](https://github.com/kobiton/automate/issues/34)). The tool returns a 200 OK with a `versionId` before the platform's app parser has determined READY vs FAILURE_PARSING. Downstream calls (`createSession`, `reserveDevice`) may then fail with no clear root cause. *Agent workaround:* after `confirmAppUpload` returns, poll `getApp(appId)` until `state` ∈ {`READY`, `FAILURE_PARSING`} before proceeding. Allow up to 60 seconds.

- **`FAILURE_PARSING` response body is empty** ([upstream #34](https://github.com/kobiton/automate/issues/34)). When an upload enters `FAILURE_PARSING`, the API response does not currently carry `parse_error`, `category`, or `message` fields. *Agent workaround:* surface the bare `FAILURE_PARSING` state to the user with a note that more detail requires checking the Kobiton portal directly.

- **`reserveDevice` conflict response lumps four failure modes** ([upstream #33](https://github.com/kobiton/automate/issues/33)). A `device_unavailable` response can mean: device is offline; device is currently utilizing; device is reserved by another user; the public-pool target is exhausted. Each implies a different retry strategy, but the current response shape does not distinguish them. *Agent workaround:* since the underlying mode is not surfaced, retrying the same device may or may not help; the safer default is to broaden the `listDevices` filter and select a different device, or surface to the user.

- **`driver.getLogs('logcat'|'browser')` silently fails on the W3C-strict endpoint** ([upstream #36](https://github.com/kobiton/automate/issues/36)). Kobiton's Appium endpoint is W3C-strict; the legacy Selenium `POST /se/log` path returns "Unsupported URI". Older WebdriverIO / Selenium client versions hit this path by default and silently lose log output. *Agent workaround:* warn the user when their test script uses `driver.getLogs()` with the legacy log API; recommend upgrading the client or switching to W3C log API.

- **Devices enter ~5 minute cleanup cooldown after `terminateSession`** ([upstream #36](https://github.com/kobiton/automate/issues/36)). During cooldown, `reserveDevice` returns the same ambiguous `device_unavailable` as the four conflict modes above. *Agent workaround:* if a `reserveDevice` call fails within 5 minutes of a `terminateSession` on the same device, treat the failure as transient cooldown and either wait 5 minutes or select a different device.

- **Per-command session data + assertion semantics not exposed** ([upstream #37](https://github.com/kobiton/automate/issues/37)). Session records advertise `execution_data.all_command_data_available: true` and `command_screenshots_available: true`, but no tool currently surfaces the per-command stream, per-command screenshots, or pass/fail assertions. This blocks the `saveTestRun` + IQS test-case CRUD use case at [upstream #24](https://github.com/kobiton/automate/issues/24). *Agent workaround:* if the user asks to "save this session as a test case", direct them to the Kobiton portal manually; no plugin-side path exists today.

- **Read-side shape divergence between `getSession` and `getSessionArtifacts`** ([upstream #35](https://github.com/kobiton/automate/issues/35)). The two endpoints use inconsistent field-naming (`device_name` vs `deviceName`, `start_time` vs `startTime`). *Agent workaround:* normalize field names when comparing data across the two endpoints.

- **`xium-portal` live-view URL asymmetry** ([upstream #35](https://github.com/kobiton/automate/issues/35)). `liveViewUrl` returns the full Portal view; `deviceOnlyViewUrl` is the same URL with `?view=device-only` appended. The asymmetry is being documented in [upstream PR #29](https://github.com/kobiton/automate/pull/29). *Agent workaround:* use `liveViewUrl` for full-chrome demos, `deviceOnlyViewUrl` for embeds or device-only sharing.

- **`listSessions` 25k-token response cap** ([upstream #35](https://github.com/kobiton/automate/issues/35)). Claude Code applies a 25k-token cap on MCP tool responses; `listSessions` responses with verbose `execution_data` per session can approach or exceed this cap. Empirically observed in the R2 audit (2026-05-04) that responses near the cap can drop fields or sessions without an explicit error surfaced to the agent. Plugin-side mitigation: default limit lowered to 10 per [tools/sessions.yaml](https://github.com/kobiton/automate/blob/main/tools/sessions.yaml). *Agent workaround:* if you need more than ~10 sessions, paginate explicitly via the `offset` parameter and verify the returned count matches expectations before treating the result as complete.

Additional gaps surfaced by the post-R3 external audit (mimosa767 on [`kobiton/automate#53`](https://github.com/kobiton/automate/issues/53), 2026-05-18), filed as upstream issues and tracked here:

- **`listSessions` silently ignores the `limit` parameter** (fork [#47](https://github.com/jeremylongshore/automate/issues/47); upstream mirror pending). Calling `listSessions(limit=N)` returns the server's default page size (20) regardless of N. Combined with the 25k-token MCP cap above, this can push responses near truncation when only a small slice was requested. *Agent workaround:* treat the returned count as authoritative; if you need a smaller result, slice the returned `sessions` array client-side. If you need a larger result, page via `offset`.

- **`getSession` response has no `has_video` indicator** (fork [#48](https://github.com/jeremylongshore/automate/issues/48); upstream mirror pending). `video_url: null` conflates "no video recorded for this session" with "video record temporarily unavailable." No boolean signal distinguishes the two. *Agent workaround:* if `video_url` is null on a `state: COMPLETE` session and the user needs to know whether video exists, call `getSessionArtifacts(sessionId)` as a secondary probe — its `video` key carries a more authoritative signal.

- **`getSessionArtifacts` does not return screenshots** (fork [#49](https://github.com/jeremylongshore/automate/issues/49); upstream mirror pending). The tool description documents four artifact categories (video, logs, screenshots, test reports). The response carries three (video, logs, testReport). Screenshots are not surfaced. *Agent workaround:* if the user asks for screenshots, explain the gap and direct them to the Kobiton portal manually. Do not fabricate a "no screenshots available" answer when the underlying API simply doesn't return them.

- **`getDeviceStatus` returns 3 fields; battery, session info, and connection state are absent** (fork [#50](https://github.com/jeremylongshore/automate/issues/50); upstream mirror pending). The tool description documents availability, current session info, battery level, and connection state. The response returns only `deviceId`, `is_booked`, `is_online`. *Agent workaround:* use `getDeviceStatus` only to answer "is this device free right now?"; for battery level, network state, or current-session detail, the only path today is to reserve the device and probe inside the session — there is no cheaper read-path.

- **`getApp.is_expired` contradicts `listApps.is_expired` for the same app id** (fork [#51](https://github.com/jeremylongshore/automate/issues/51); upstream mirror pending). For the same app id in the same minute, `listApps` and `getApp` can return opposite boolean values for `is_expired`. `listApps` also carries the actual `expiry_date` timestamp; `getApp` drops it. *Agent workaround:* trust `listApps.latest_version.is_expired` as the authoritative source for expiry decisions. Do not gate uploads or session creation on `getApp.is_expired`.

- **`uploadAppToStore` response carries v1/v2 doc inconsistency** (fork [#52](https://github.com/jeremylongshore/automate/issues/52); upstream mirror pending). The response's `confirm_upload.description` references `/v2/apps` while `confirm_upload.path` references `/v1/apps`. The actual `confirmAppUpload` tool doesn't take a version parameter, so this is downstream-consumer confusion rather than a hard failure. *Agent workaround:* none needed — call `confirmAppUpload` with the `appPath` and `filename` from the response per the tool description; ignore the v1/v2 strings in the `confirm_upload` sub-object.

For each finding above, the full evidence + repro + architectural-fix proposal lives on the linked issue.
