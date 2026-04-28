# Capabilities reference

This document is the reference for the capability fields the `run-automation-suite` skill parses from local Appium test scripts and reconciles against Kobiton device-cloud requirements.

The skill body in `SKILL.md` orchestrates the parsing flow; this reference catalogs the field set, the per-field comparison policy, and the Appium-runtime special-case handling. Loaded on-demand by Claude when capability-detail context is needed; not loaded as part of the default skill body.

## Capability fields parsed from the user's script

The skill extracts the following capability values from the test script source code:

| Field | Notes |
|---|---|
| `platformName` | Android / iOS |
| `udid` | hardcoded or parameterized |
| `app` | app URL or kobiton-store reference |
| `sessionName`, `sessionDescription` | Kobiton session metadata |
| `automationName` | UiAutomator2, XCUITest, etc. |
| `browserName` | if browser-based test |
| `deviceOrientation` | portrait / landscape |
| Any `kobiton:*` vendor extensions | especially `kobiton:runtime` (see special-case section below) |

The skill identifies how the UDID is passed into the script (CLI argument, environment variable, or hardcoded) so it can be overridden with the device the user selected in workflow Step 2.

## Per-field comparison policy after rendering

After running `scripts/render-capabilities.js`, the skill compares the rendered JSON capabilities against the parsed script capabilities and applies one of three policies per field:

### Must-match (auto-edit)

Fields: `platformName`, `platformVersion`, `appium:udid`, `appium:deviceName`, `appium:app` / `browserName`, `appium:automationName`.

If different from the rendered output: show what will change and edit the script automatically. These must match the selected device and app — divergence will break the run.

### Suggested defaults (ask first)

Fields: `kobiton:sessionName`, `kobiton:sessionDescription`, `kobiton:deviceOrientation`, `kobiton:captureScreenshots`, `appium:noReset`, `appium:fullReset`.

If different from the rendered output or missing: show the diff and ask the user before changing. The user may have intentionally set different values.

### User-controlled (leave untouched)

Any capabilities in the user's script that are not in the rendered output. Never inject or modify `kobiton:runtime` unless the user explicitly asks (see special-case below).

## Appium runtime — special-case handling

Check if the script contains `'kobiton:runtime': 'appium'` or equivalent.

- If it does **NOT**, do not inject it. The default Kobiton runtime will be used.
- Only if the user explicitly asks to use the Appium runtime should `'kobiton:runtime': 'appium'` be added to the script's capabilities.

This is a deliberate opt-in — silent injection of `kobiton:runtime` would change which test runner Kobiton dispatches and may produce surprising results for users who were relying on the default.

## Related references

- `references/templates/appium.ejs` — the EJS template `render-capabilities.js` consumes to emit the rendered capabilities JSON
- `scripts/render-capabilities.js` — the renderer (the only runtime code in this skill); reads the template, applies CLI flags + defaults, emits JSON
- `scripts/render-capabilities.test.js` — vitest suite for the renderer
