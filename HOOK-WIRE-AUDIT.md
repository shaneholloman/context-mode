# Codex CLI Hook Wire Protocol Audit

**Date:** 2026-04-03
**Codex source:** `codex-rs/hooks/src/schema.rs`, `codex-rs/hooks/src/engine/output_parser.rs`
**Our hooks:** `hooks/codex/{pretooluse,posttooluse,sessionstart}.mjs`, `hooks/core/formatters.mjs`

---

## 1. PreToolUse

### Stdin (Codex sends to hook)

Fields from `PreToolUseCommandInput` (no `rename_all` — fields are snake_case):

| Field | Type | Our hook reads |
|---|---|---|
| `session_id` | String | Yes (via `getSessionId`) |
| `turn_id` | String | No (not used) |
| `transcript_path` | String\|null | No (not used) |
| `cwd` | String | Yes (via `getInputProjectDir`) |
| `hook_event_name` | "PreToolUse" | No (not used) |
| `model` | String | No (not used) |
| `permission_mode` | String | No (not used) |
| `tool_name` | String | **Yes** — `input.tool_name` (line 22) |
| `tool_input` | `{command: String}` | **Yes** — `input.tool_input` (line 23) |
| `tool_use_id` | String | No (not used) |

**Note:** `tool_input` sub-struct has `rename_all = "camelCase"` but the only field is `command` (same in both cases). No mismatch.

### Stdout (hook sends to Codex)

Codex parses `PreToolUseCommandOutputWire` (`rename_all = "camelCase"` on nested structs):

| Field | Wire name | Supported | Our hook sends |
|---|---|---|---|
| `universal.continue` | `continue` | **REJECTED** if false (unsupported) | Not sent (correct) |
| `universal.stop_reason` | `stopReason` | **REJECTED** (unsupported) | Not sent (correct) |
| `universal.suppress_output` | `suppressOutput` | **REJECTED** if true (unsupported) | Not sent (correct) |
| `universal.system_message` | `systemMessage` | Supported | Not sent (ok) |
| `decision` | `decision` | `"approve"` **REJECTED**, `"block"` requires reason | Not sent (correct) |
| `reason` | `reason` | **REJECTED** without decision | Not sent (correct) |
| `hookSpecificOutput.hookEventName` | `hookEventName` | Required (not `#[serde(default)]`) | **Yes** — `"PreToolUse"` |
| `hookSpecificOutput.permissionDecision` | `permissionDecision` | `"deny"` supported (needs reason), `"allow"` **REJECTED**, `"ask"` **REJECTED** | **Yes** — `"deny"` only |
| `hookSpecificOutput.permissionDecisionReason` | `permissionDecisionReason` | Required with deny | **Yes** — sends reason |
| `hookSpecificOutput.updatedInput` | `updatedInput` | **REJECTED** (unsupported) | Not sent (correct, returns null) |
| `hookSpecificOutput.additionalContext` | `additionalContext` | **REJECTED** (unsupported) | Not sent (correct, returns null) |

**Verdict: PASS**

Our deny output: `{hookSpecificOutput: {hookEventName: "PreToolUse", permissionDecision: "deny", permissionDecisionReason: "..."}}` — matches wire format exactly. `ask`, `modify`, `context` correctly return `null` (passthrough).

---

## 2. PostToolUse

### Stdin (Codex sends to hook)

Fields from `PostToolUseCommandInput` (no `rename_all` — snake_case):

| Field | Type | Our hook reads |
|---|---|---|
| `session_id` | String | Yes (via `getSessionId`) |
| `turn_id` | String | No (not used) |
| `transcript_path` | String\|null | No (not used) |
| `cwd` | String | Yes (via `getInputProjectDir`) |
| `hook_event_name` | "PostToolUse" | No (not used) |
| `model` | String | No (not used) |
| `permission_mode` | String | No (not used) |
| `tool_name` | String | **Yes** — `input.tool_name` (line 37) |
| `tool_input` | `{command: String}` | **Yes** — `input.tool_input` (line 38) |
| `tool_response` | Value (JSON) | **Yes** — `input.tool_response` (line 39-41) |
| `tool_use_id` | String | No (not used) |

**Note:** Our hook reads `tool_response` and coerces to string if not already. The Codex wire sends it as `Value` (arbitrary JSON). Our coercion is safe.

### Stdout (hook sends to Codex)

Codex parses `PostToolUseCommandOutputWire`:

| Field | Wire name | Supported | Our hook sends |
|---|---|---|---|
| `universal.continue` | `continue` | Supported | Not sent (defaults to true, correct) |
| `universal.stop_reason` | `stopReason` | Supported | Not sent (ok) |
| `universal.suppress_output` | `suppressOutput` | **REJECTED** if true (unsupported) | Not sent (correct) |
| `universal.system_message` | `systemMessage` | Supported | Not sent (ok) |
| `decision` | `decision` | Only `"block"` (requires reason) | Not sent (correct — hook is observation-only) |
| `reason` | `reason` | Only with decision:block | Not sent (correct) |
| `hookSpecificOutput.hookEventName` | `hookEventName` | Required | Not sent (hook outputs nothing) |
| `hookSpecificOutput.additionalContext` | `additionalContext` | Supported | Not sent |
| `hookSpecificOutput.updatedMCPToolOutput` | `updatedMCPToolOutput` | Supported | Not sent |

**Verdict: PASS**

Our PostToolUse hook is observation-only (captures session events). It produces **no stdout**, which Codex handles as a no-op passthrough. Correct behavior.

---

## 3. SessionStart

### Stdin (Codex sends to hook)

Fields from `SessionStartCommandInput` (no `rename_all` — snake_case):

| Field | Type | Our hook reads |
|---|---|---|
| `session_id` | String | Yes (via `getSessionId`) |
| `transcript_path` | String\|null | No (not used) |
| `cwd` | String | Yes (via `getInputProjectDir`) |
| `hook_event_name` | "SessionStart" | No (not used) |
| `model` | String | No (not used) |
| `permission_mode` | String | No (not used) |
| `source` | String ("startup"\|"compact"\|"resume") | **Yes** — `input.source` (line 40) |

### Stdout (hook sends to Codex)

Codex parses `SessionStartCommandOutputWire`:

| Field | Wire name | Supported | Our hook sends |
|---|---|---|---|
| `universal.continue` | `continue` | Supported | Not sent (defaults true, ok) |
| `universal.stop_reason` | `stopReason` | Supported | Not sent (ok) |
| `universal.suppress_output` | `suppressOutput` | Supported | Not sent (ok) |
| `universal.system_message` | `systemMessage` | Supported | Not sent (ok) |
| `hookSpecificOutput.hookEventName` | `hookEventName` | Required (no `#[serde(default)]`) | **NOT SENT** |
| `hookSpecificOutput.additionalContext` | `additionalContext` | Supported | **Yes** — sends the routing block |

**Our output (line 96-98):**
```json
{"hookSpecificOutput": {"additionalContext": "..."}}
```

**Codex expects:**
```json
{"hookSpecificOutput": {"hookEventName": "SessionStart", "additionalContext": "..."}}
```

### FINDING: `hookEventName` is MISSING from SessionStart output

`SessionStartHookSpecificOutputWire` has `pub hook_event_name: HookEventNameWire` with **no `#[serde(default)]`** annotation. Under `#[serde(deny_unknown_fields)]` + `rename_all = "camelCase"`, this means `hookEventName` is a **required field** for deserialization.

However, `HookEventNameWire` derives `Deserialize` and the enum has `Default` if it derives it — let me check... It does NOT derive `Default`. This means the missing `hookEventName` field will cause a **deserialization error**.

**BUT** — the outer `hook_specific_output` field on `SessionStartCommandOutputWire` has `#[serde(default)]` and is `Option<...>`. So if Codex's JSON parser encounters an error deserializing `hookSpecificOutput`, the behavior depends on whether serde treats it as a missing-field-within-object error or propagates it.

**Actual risk:** Since `hookSpecificOutput` is `Option<T>` with `#[serde(default)]`, providing a malformed value (object missing required field) will cause a **parse failure** of the entire output, not just the nested struct. The `parse_json` function in output_parser.rs returns `None` on parse failure, meaning the hook output is **silently ignored**.

**Verdict: FAIL**

The `additionalContext` (routing block) is silently dropped because `hookEventName` is missing from our `hookSpecificOutput` object.

---

## Summary

| Hook | Input Fields | Output Fields | Verdict |
|---|---|---|---|
| **PreToolUse** | PASS — reads `tool_name`, `tool_input`, `cwd`, `session_id` correctly | PASS — `deny` format matches wire exactly | **PASS** |
| **PostToolUse** | PASS — reads `tool_name`, `tool_input`, `tool_response`, `cwd`, `session_id` correctly | PASS — produces no output (observation-only) | **PASS** |
| **SessionStart** | PASS — reads `source`, `session_id`, `cwd` correctly | **FAIL** — missing required `hookEventName` in `hookSpecificOutput` | **FAIL** |

## Required Fix

File: `/Users/mksglu/Server/Mert/context-mode/hooks/codex/sessionstart.mjs` (line 96-98)

**Current:**
```js
process.stdout.write(JSON.stringify({
  hookSpecificOutput: { additionalContext },
}) + "\n");
```

**Should be:**
```js
process.stdout.write(JSON.stringify({
  hookSpecificOutput: { hookEventName: "SessionStart", additionalContext },
}) + "\n");
```

---

## Source References

- `codex-rs/hooks/src/schema.rs` — `SessionStartHookSpecificOutputWire` struct (no `#[serde(default)]` on `hook_event_name`)
- `codex-rs/hooks/src/engine/output_parser.rs` — `parse_session_start()` function, `parse_json()` returns None on failure
- `hooks/codex/sessionstart.mjs` lines 96-98 — missing `hookEventName`
- `hooks/core/formatters.mjs` codex section — PreToolUse deny correctly includes `hookEventName`
