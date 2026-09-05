# @open-design/dsh-runtime stdio protocol

> Status: **baseline** (adopted by `@open-design/dsh-runtime`, owned here)
> Applies to: wire between the Open Design daemon (host) and
> `dsh --profile open-design {--probe|--models|--stdio}` (profile adapter,
> DeepSeek Harness + this bundle). Companion schema:
> `dsh-stdio-contract.schema.json`.
> Implementation source of truth: `src/protocol.ts`, `src/startup.ts`,
> `src/index.ts`; contract tests in `tests/protocol.test.ts`.

## 0. Contract

OpenDesign starts one short-lived `dsh --profile open-design <mode>` child
process per run and exchanges one JSON object per line over stdin/stdout
(JSONL). The child never writes anything except protocol frames to stdout.
This document freezes the wire meaning so Open Design (host) and DeepSeek
Harness (profile adapter) can release independently.

## 1. Participants and lifecycle

| Side | Role | Process model |
|---|---|---|
| Open Design daemon | host | spawns one short-lived child per design run |
| `dsh --profile open-design` | profile adapter | exits after exactly one run; cold resume across processes via Harness session storage |

Startup modes (exactly one required; enforced by `src/startup.ts`):

| Mode | Child action |
|---|---|
| `--probe` | print one `probe` frame, exit 0 |
| `--models` | print one `models` frame (model catalog), exit 0 |
| `--stdio` | print one `ready` frame, then serve host commands until the run terminates, managed exit 0 with a bounded 1 s process-exit fallback |

`cordis.patch.yml` adds only the OD protocol boundary on top of the official
`dsh-base` composition. The user's Harness installation owns credentials,
settings, tools, sessions, and provider adapters. Setup is explicit on the
first incompatible selection; cancelling does not select the agent or mutate
the Harness profile.

## 2. Envelope and identity

Every frame is an object with `v` and `type`. Identity frames also carry
`runtime`, `protocol_version`, `plugin_version`, and `capabilities`:

```json
{ "v": 1, "type": "probe", "runtime": "open-design",
  "protocol_version": 1, "plugin_version": "0.1.0",
  "capabilities": { "session_resume": true, "session_cancel": true, "structured_events": true } }
```

- `runtime` is always `open-design`.
- `protocol_version` (currently 1) is the wire version; bump only on a breaking
  change.
- `plugin_version` is the `@open-design/dsh-runtime` version; hosts must
  tolerate any `plugin_version`.
- `capabilities` is a static advertisement. Adding a capability is additive;
  removing or reinterpreting one is a breaking change.

The `ready` frame has the same shape with `type: "ready"` and is the first
frame in `--stdio` mode, before any host command is accepted.

## 3. Host-to-child commands (stdin, one per line)

### execute

```json
{ "v": 1, "type": "execute", "request_id": "<host-unique>",
  "cwd": "<project-abs-path>", "prompt": "<task text>",
  "mcp_servers": [],
  "resume_session_id": "<optional>",
  "model": { "provider": "<provider-id>", "id": "<model-id>" },
  "reasoning_effort": "<optional string>" }
```

- `request_id` is non-empty and host-unique for correlation.
- `mcp_servers` is required; it may be an empty array.
- `resume_session_id`: when present and non-empty, resume that Harness session
  instead of creating one. The default new-session id is `od-<uuid>`.
- `model`: optional full selection; absence means the profile default model.
- Unknown fields are dropped by the child; extra fields never change the
  meaning of a v1 execute.

### cancel

```json
{ "v": 1, "type": "cancel", "request_id": "<matching execute request_id>" }
```

Cancellation is latched: a cancel received before the matching execute is
active (spawn/handshake window) is replayed as soon as that execute activates.
Hosts expect at most one terminal `result` per `request_id`.

### Invalid or busy lines

- An unparseable or invalid line yields `protocol_error`
  (`DSH_PROFILE_INVALID_COMMAND`); the child keeps reading.
- A second `execute` on the same process yields `protocol_error`
  (`DSH_PROFILE_BUSY`); one process accepts exactly one execute.
- Empty lines are ignored.

## 4. Child-to-host frames (stdout)

### session

```json
{ "v": 1, "type": "session", "request_id": "...", "session_id": "od-...",
  "resumed": false }
```

### text / thinking

```json
{ "v": 1, "type": "text", "request_id": "...", "content": "<text-delta>" }
{ "v": 1, "type": "thinking", "request_id": "...", "content": "<reasoning-delta>" }
```

Empty deltas are not emitted.

### tool_call / tool_result

```json
{ "v": 1, "type": "tool_call", "request_id": "...", "call_id": "...",
  "name": "...", "arguments": { } }
{ "v": 1, "type": "tool_result", "request_id": "...", "call_id": "...",
  "name": "<tool or 'tool'>", "output": "<concatenated text>", "is_error": false }
```

### usage

```json
{ "v": 1, "type": "usage", "request_id": "...", "provider": "...",
  "model": "...", "input_tokens": 1, "output_tokens": 1,
  "cache_read_tokens": 1, "cache_write_tokens": 1 }
```

`cache_read_tokens` and `cache_write_tokens` are optional.

### result (exactly one terminal frame per execute)

```json
{ "v": 1, "type": "result", "request_id": "...", "session_id": "od-...",
  "status": "completed", "output": "<full assistant text, optional>",
  "stop_reason": "completed", "resume_rejected": false,
  "error": { "code": "...", "message": "..." } }
```

- `status`: `completed` | `cancelled` | `failed`.
- `stop_reason`: the raw Harness turn-end kind (`completed`, `max-tokens`,
  `aborted`, unknown kinds pass through unchanged).
- `output` is omitted when empty.
- `error` is present only when `status` is `failed`.
- `resume_rejected` is `true` only for a failed resume; that `result` still
  carries `session_id`.

### protocol_error

```json
{ "v": 1, "type": "protocol_error", "request_id": "<optional>",
  "code": "DSH_PROFILE_INVALID_COMMAND", "message": "..." }
```

## 5. Status and error-code contract

Turn-end to wire mapping (do not change; hosts rely on it):

| Harness turn reason | wire status | notes |
|---|---|---|
| `completed`, `max-tokens` | `completed` | |
| `aborted` | `cancelled` | user/model cancellation |
| `error` | `failed` | passthrough code, else `DSH_PROFILE_TURN_FAILED` |
| `blocked` | `failed` | `DSH_PROFILE_TURN_BLOCKED` |
| idle with no turn end | `failed` | `DSH_PROFILE_MISSING_TURN_END` |

Error `code`: the child passes through a structured Harness error `code` when
it is a non-empty string, otherwise uses its own `DSH_PROFILE_*` code:
`INVALID_COMMAND`, `BUSY`, `SESSION_CREATE_FAILED`, `RESUME_REJECTED`,
`EXECUTION_FAILED`, `TURN_FAILED`, `TURN_BLOCKED`, `MISSING_TURN_END`.
`message` is always a safe, non-empty human string.

## 6. Versioning and compatibility rules

1. Additive only within a wire version: new frame types, new optional fields,
   and new capability flags are allowed. Renaming, removing, or reinterpreting
   existing fields, types, or values within `v` is not.
2. Unknown-field policy: host-to-child unknown fields are dropped by the child;
   child-to-host, hosts must ignore unknown frame types and unknown optional
   fields (forward compatibility for older hosts).
3. A breaking change (rename/remove/retype a field or frame, change the status
   mapping, remove a capability) requires a `protocol_version` bump, new golden
   replay pairs on both sides, and a coordinated release.
4. `plugin_version` or DeepSeek Harness version changes alone never imply wire
   breakage.
5. One logical run equals one child process equals one `execute` equals one
   terminal `result`; `session_id` is the only cross-process resume token
   (cold resume).
6. Both sides keep the protocol under golden/replay tests so a drift fails the
   owning CI (see next section).

## 7. Testing obligations

- Open Design: keep contract tests in `tests/protocol.test.ts` and grow a
  golden corpus (probe, models, execute through session/text/tool/usage/result,
  cancel, invalid line, busy).
- DeepSeek Harness: consume the built `open-design-dsh-runtime-<v>.tgz` in a
  keyless recorded-session replay through the `open-design` profile.
- Maintain a compatibility matrix (checked in CI or documented at each
  release): DeepSeek Harness version x dsh-runtime version x OpenDesign app
  version.

## 8. Current implementation reference

| Item | Value |
|---|---|
| `protocol_version` | 1 |
| dsh-runtime version | 0.1.0 |
| DSH packages consumed | `@deepseek-ai/dsh-cmdline@0.1.1-rc.2` (dependency); peers `@deepseek-ai/{cordis,cordis-plugin-loader,dsh-agent,dsh-agent-default-model,dsh-invariants,dsh-llm,dsh-session} >= 0.1.1-rc.0` |
| frame direction | host-to-child: stdin lines; child-to-host: stdout lines only; stderr is for human diagnostics |
| process-exit fallback | 1 s after managed exit; the host keeps a ~3 s cancellation grace |

## 9. Resolved decisions (baseline, adopted 2026-09-05)

1. **Host forward compatibility.** The host MUST ignore unknown frame types
   and unknown optional fields, and validate only known required fields
   strictly. This mirrors the child's behavior of dropping unknown
   host-command fields, so both directions tolerate additive evolution within
   a wire version. (Implementation note: the current host adapter in
   `apps/daemon/src/agent-protocol/dsh-profile/` is strict; aligning it with
   this rule is tracked as a host-side change.)
2. **Model catalog is a runtime query result.** A host MUST query the catalog
   (`--models` or equivalent) per run and MUST NOT assume a fixed provider or
   model set, nor that a catalogued model still exists at execute time.
   Selection references catalogued ids and fails cleanly when a referenced id
   is absent.
3. **Tail-output truncation (open verification).** Recorded as a known risk
   pending verification. Before any protocol change, run a large-output test
   (output that continues past the ~1 s exit-fallback window) to verify no
   truncation; only if truncation is reproduced, add a drain/acknowledgement
   mechanism in a later wire version.
4. **Platform grading in the compatibility matrix.** macOS and Linux are the
   formally supported replay platforms; Windows native is best-effort. A
   Windows-only failure must block only Windows-specific correctness and must
   never be reported as protocol drift. The matrix records the platform next
   to each Harness x runtime version pair.
