# Post-R3 tool-description doc-drift catalog (F45-F50)

> **Status:** captured 2026-05-18 from a post-R3 empirical-probe audit triggered by Stephen Penn's external audit reply on [`kobiton/automate#53`](https://github.com/kobiton/automate/issues/53#issuecomment-4476188526). Each finding below was probed against the live `api.kobiton.com/mcp` endpoint; raw payload archives are on file with Intent Solutions and available on request.
>
> **What this file is:** a catalog of points where the current `tools/*.yaml` descriptions diverge from the actual MCP server response shape. Each finding is filed as a closed upstream audit issue (`#55-#60`). The descriptions in `tools/*.yaml` have been narrowed to match server reality where conservative; the longer narrative + repro detail for each finding lives here.
>
> **Out of scope for this file:** the server-side response-shape changes themselves (whether the server should add a `has_video` field, surface `screenshots` artifacts, etc.) are folded into the consolidated [`docs/server-side-recommendations.md`](server-side-recommendations.md). This file documents *what consumers will observe today*, not *what the server should change*.

## Contents

| Finding | Tool | Symptom | Closed audit issue |
|---|---|---|---|
| [F45](#f45--listsessionslimit-silently-ignored) | `listSessions` | `limit` parameter is accepted but server returns its default page size regardless | [`#55`](https://github.com/kobiton/automate/issues/55) |
| [F46](#f46--getsession-response-has-no-has_video-indicator) | `getSession` | No `has_video` field exists in the response; `video_url` may be null | [`#56`](https://github.com/kobiton/automate/issues/56) |
| [F47](#f47--getsessionartifacts-omits-screenshots-category) | `getSessionArtifacts` | Description lists "screenshots" but response payload omits this category | [`#57`](https://github.com/kobiton/automate/issues/57) |
| [F48](#f48--getdevicestatus-returns-3-fields-description-lists-more) | `getDeviceStatus` | Returns `deviceId` + `is_booked` + `is_online` only; description references battery, current-session info, connection state | [`#58`](https://github.com/kobiton/automate/issues/58) |
| [F49](#f49--getappis_expired-contradicts-listappsis_expired) | `getApp` / `listApps` | For the same app id, `getApp.is_expired` disagrees with `listApps.is_expired` | [`#59`](https://github.com/kobiton/automate/issues/59) |
| [F50](#f50--uploadapptostoreconfirm_upload-carries-v1v2-contradiction) | `uploadAppToStore` | `confirm_upload.description` references `/v2/apps`; `confirm_upload.path` references `/v1/apps` | [`#60`](https://github.com/kobiton/automate/issues/60) |

---

## F45 — `listSessions.limit` silently ignored

**Closed audit issue:** [`kobiton/automate#55`](https://github.com/kobiton/automate/issues/55)

### Symptom

`listSessions({ limit: 3, userIntent: "..." })` returns 20 sessions. The `limit` request parameter is accepted without error but does not constrain the result count — the server returns its default page size (`current_size: 20`) regardless of the value passed.

### Plugin-side mitigation in this PR

`tools/sessions.yaml` `listSessions.inputSchema.properties.limit.description` was already expanded by PR-C to flag the 25k-token cap rationale and direct consumers to verify response size. This file documents the underlying server-side silent-ignore behavior for consumer reference.

### Server-side recommendation

Honor the request-side `limit` parameter, or surface an explicit error if the value is unsupported. Captured in [`docs/server-side-recommendations.md`](server-side-recommendations.md).

---

## F46 — `getSession` response has no `has_video` indicator

**Closed audit issue:** [`kobiton/automate#56`](https://github.com/kobiton/automate/issues/56)

### Symptom

External audit (mimosa767) reported `video_url: null` while `has_video: true` — an internal contradiction. Our probe found that the `getSession` response does not include a `has_video` field at all. Two probed sessions returned different `video_url` values (one populated, one null) with no indicator field to disambiguate "no video for this session" from "video field absent from response."

Response keys observed on probed `COMPLETE` sessions: `id, name, state, type, automation_type, device_name, platform_name, platform_version, is_cloud, is_crashed, created_at, ...` plus `video_url` (may be null).

### Plugin-side mitigation in this PR

`tools/sessions.yaml` `getSession.description` clarifies that the response carries `video_url` (which may be null) and that no separate `has_video` indicator field is provided.

### Server-side recommendation

Add an explicit `has_video` boolean or change `video_url` semantics so `null` reliably means "no video exists for this session." Captured in [`docs/server-side-recommendations.md`](server-side-recommendations.md).

---

## F47 — `getSessionArtifacts` omits `screenshots` category

**Closed audit issue:** [`kobiton/automate#57`](https://github.com/kobiton/automate/issues/57)

### Symptom

The `getSessionArtifacts` tool description currently advertises four artifact categories: **video recording, device logs, screenshots, and test reports**. The actual response payload returns three of four — the `screenshots[]` category is not surfaced anywhere.

### Plugin-side mitigation in this PR

`tools/sessions.yaml` `getSessionArtifacts.description` is amended to drop the "screenshots" mention so the description matches the empirically-observed response shape.

### Server-side recommendation

Either surface the `screenshots[]` array in the response (matching the implied advertisement) or document that screenshots are accessible only via a separate endpoint. Captured in [`docs/server-side-recommendations.md`](server-side-recommendations.md).

---

## F48 — `getDeviceStatus` returns 3 fields; description lists more

**Closed audit issue:** [`kobiton/automate#58`](https://github.com/kobiton/automate/issues/58)

### Symptom

The `getDeviceStatus` tool description currently advertises that it returns "real-time status of a specific device including availability, current session info, battery level, and connection state."

Empirical response on a probed device:

```json
{ "deviceId": 1884315031, "is_booked": false, "is_online": true }
```

Three fields total. The mapping against the advertised categories:

| Documented category | In response? | Notes |
|---|---|---|
| availability | covered by `is_booked` + `is_online` | ✓ |
| current session info | absent | no `session_id` / `active_session` / similar field surfaced |
| battery level | absent | no `battery_level` field |
| connection state | absent | no `network_type` / `signal_strength` field |

### Plugin-side mitigation in this PR

`tools/devices.yaml` `getDeviceStatus.description` is narrowed to describe the three fields actually returned (`is_booked`, `is_online`).

### Server-side recommendation

Either add the advertised fields to the response or narrow the server-side tool description to match the empirical shape. Captured in [`docs/server-side-recommendations.md`](server-side-recommendations.md).

---

## F49 — `getApp.is_expired` contradicts `listApps.is_expired`

**Closed audit issue:** [`kobiton/automate#59`](https://github.com/kobiton/automate/issues/59)

### Symptom

For the same app id, in the same minute, `listApps` and `getApp` disagree on `is_expired`. The discrepancy was probed against app id 678538 (`iOSPerf`) on 2026-05-18:

**`listApps` response** (filtered to app id 678538):

```
latest_version.is_expired:  true
latest_version.expiry_date: 2026-02-27T08:03:34.695Z   (~3 months past)
```

**`getApp` response** (same app id):

```
latest_version.is_expired:  false   ← contradicts listApps
latest_version.expiry_date: (field dropped)
```

`listApps` carries the canonical expiry timestamp and the consistent boolean; `getApp` reports the app as not expired and omits the timestamp entirely.

### Plugin-side mitigation in this PR

`tools/apps.yaml` `getApp.description` notes that `is_expired` may disagree with `listApps` for the same app id and directs consumers to treat `listApps`'s `expiry_date` timestamp as canonical. `tools/apps.yaml` `listApps.description` notes that the `expiry_date` it returns is the canonical timestamp.

### Server-side recommendation

Either compute `is_expired` consistently across both endpoints (using the same `expiry_date` and "now") or document a single canonical endpoint for expiry. Captured in [`docs/server-side-recommendations.md`](server-side-recommendations.md).

---

## F50 — `uploadAppToStore.confirm_upload` carries v1/v2 contradiction

**Closed audit issue:** [`kobiton/automate#60`](https://github.com/kobiton/automate/issues/60)

### Verification posture

mimosa767's external-audit report captured the contradiction from a live upload session. Intent Solutions deliberately did not trigger a destructive re-probe — uploading a real APK/IPA creates a partner-system record + S3 file that would then need cleanup, and the original report is concrete enough to file on.

### Symptom

The `uploadAppToStore` response contains an internal documentation inconsistency in the `confirm_upload` object:

- `confirm_upload.description` references `POST /v2/apps`
- `confirm_upload.path` references `/v1/apps`

Both fields describe the same confirm step in the same response payload. Whichever consumer reads the response first will get two contradictory hints about which API version to call next.

### Plugin-side mitigation in this PR

`tools/apps.yaml` `uploadAppToStore.description` notes that the response's `confirm_upload` field may carry contradictory v1 vs v2 endpoint hints, and that consumers should treat `confirm_upload.path` as the canonical confirm endpoint (since `path` is what an HTTP client actually issues).

### Server-side recommendation

Align both `confirm_upload.description` and `confirm_upload.path` to the same API version (preferably v2, dropping the v1 reference entirely). Captured in [`docs/server-side-recommendations.md`](server-side-recommendations.md).

---

## How to read this catalog

Each finding above maps 1:1 to a closed `kobiton/automate` audit issue (`#55-#60`). The plugin-side `tools/*.yaml` description edits in this PR are conservative — they make the descriptions match what the server actually returns today, so consumers calling these tools see accurate documentation without depending on a server-side change shipping first.

The server-side response-shape changes are folded into [`docs/server-side-recommendations.md`](server-side-recommendations.md) (separate close-out PR), where they sit alongside the broader R2 + R3 server-side recommendations.

- Jeremy Longshore
intentsolutions.io
