# oc-agent CLI Specification

## Overview
`oc-agent` is a standalone CLI that spawns and manages a daemonized opencode ACP client for a given working directory. It is designed for machine consumption (e.g., clawdbot) and outputs JSON to stdout for every command. Logs are written to stderr.

Core goals:
- Provide a stable CLI for opening and controlling opencode ACP sessions.
- Keep session lifecycle explicit and non-interactive.
- Support streaming prompt outputs by default.
- Run as a daemon that can be discovered and reconnected by session ID.

## Transport & Process Model
- `opencode acp` is started as a **stdio** subprocess with `--cwd` set to the target directory.
- `oc-agent start` daemonizes and keeps the ACP connection open.
- `oc-agent prompt` connects to the daemon via IPC using named pipes (Windows) or Unix sockets (POSIX), using the same Node.js `net` API.
- If a daemon is not running for a given session, `prompt --session` **restarts** the daemon and reconnects.

## Registry & State
Registry stores session-to-daemon mapping.
- POSIX: `$XDG_STATE_HOME/oc-agent/` (fallback `~/.local/state/oc-agent/`)
- Windows: `%LOCALAPPDATA%\oc-agent\`

Registry entries include:
- `sessionId`
- `cwd`
- `pipePath`
- `pid`
- `lastActiveAt`

## Output Format
All commands write JSON to stdout.

Envelope:
```json
{ "ok": true, "data": { /* command result */ } }
{ "ok": false, "error": { "code": "SOME_CODE", "message": "Human readable" } }
```

Exit codes:
- `0` on success
- `1` on error (stdout still returns JSON error envelope)

## Global Flags
- `--version` (root only): prints version from `package.json`.

Per-command flags:
- `--help`
- `--log-level <debug|info|warn|error>` (default `info`)

## Commands

### `start`
Starts or discovers a session for the current working directory and daemonizes an ACP connection.

Flags:
- `--cwd <path>`: absolute path for the session.
- `--new`: always create a new session.
- `--continue`: continue the most recent session.
- `--session <id>`: attach to a specific session.
- `--idle-timeout <minutes>`: overrides idle timeout (default 20).

Environment:
- `OC_AGENT_IDLE_TIMEOUT` (minutes)

Behavior:
1. Run `opencode session list --format json` scoped to `cwd`.
2. If no sessions exist and no flags are provided, create a new session.
3. If one or more sessions exist **and no flags are provided**, return a non-interactive response listing sessions and the command options the caller should invoke.
4. If `--new`, create a new session.
5. If `--continue`, attach to most recent session.
6. If `--session`, attach to that session.

Output (success):
```json
{
  "ok": true,
  "data": {
    "action": "started",
    "sessionId": "sess_...",
    "cwd": "/abs/path",
    "pipePath": "...",
    "pid": 12345
  }
}
```

Output (sessions exist, decision required):
```json
{
  "ok": true,
  "data": {
    "action": "select_session",
    "sessions": [
      { "sessionId": "sess_...", "title": "...", "updatedAt": "..." }
    ],
    "suggestions": {
      "new": "oc-agent start --new --cwd /abs/path",
      "continue": "oc-agent start --continue --cwd /abs/path",
      "session": "oc-agent start --session=... --cwd /abs/path"
    }
  }
}
```

### `prompt`
Send a prompt to an existing session via the daemon.

Input sources (priority order):
1. `--prompt "text"`
2. First positional argument
3. stdin

Flags:
- `--session <id>` (required)
- `--wait-for-turn-finish` (optional): buffer all updates and return only when `session/prompt` completes
- `--auto-approve` (optional): auto-approve permission requests

Default behavior (streaming):
- Each `session/update` notification is emitted as a **single JSON line** (NDJSON) with the envelope.
- Final line is the `session/prompt` response (with `stopReason`).

Output (streaming line example):
```json
{ "ok": true, "data": { "sessionId": "sess_...", "type": "session/update", "payload": { /* update */ } } }
```

Output (wait-for-finish):
```json
{
  "ok": true,
  "data": {
    "sessionId": "sess_...",
    "updates": [ /* ordered session/update payloads */ ],
    "result": { "stopReason": "end_turn" }
  }
}
```

### `respond-permission`
Respond to a `session/request_permission` received during a prompt.

Flags:
- `--session <id>`
- `--request-id <id>`
- `--allow` | `--deny`

### `cancel`
Cancel an in-flight prompt.

Flags:
- `--session <id>`

### `stop`
Stop the daemon for a session.

Flags:
- `--session <id>`

### `list`
List active sessions known to `oc-agent`.

Output:
```json
{ "ok": true, "data": { "sessions": [ { "sessionId": "...", "cwd": "...", "pid": 123 } ] } }
```

### `status`
Check if a session daemon is alive.

Flags:
- `--session <id>`

Output:
```json
{ "ok": true, "data": { "sessionId": "...", "alive": true } }
```

## Idle Timeout
- Default: 20 minutes
- Configurable via `--idle-timeout` or `OC_AGENT_IDLE_TIMEOUT`
- Daemon exits when idle timeout is exceeded or when opencode subprocess exits unexpectedly.

## Capabilities
- `oc-agent` **does not** advertise ACP client capabilities for `fs` or `terminal`.
- No MCP server configuration is supported in the MVP.

## Named Pipe Paths
- POSIX: use Unix socket path in the registry (e.g., `/tmp/oc-agent-<session>.sock`)
- Windows: `\\.\pipe\oc-agent-<session>`

## Error Codes (Initial Set)
- `SESSION_NOT_FOUND`
- `DAEMON_NOT_RUNNING`
- `OPENCODE_NOT_FOUND`
- `INVALID_PROMPT`
- `INVALID_ARGUMENTS`
- `PERMISSION_REQUEST_NOT_FOUND`
- `INTERNAL_ERROR`

## Open Questions
- Exact JSON schema for `session/update` and `session/request_permission` payloads (pass-through vs normalized)
- Whether to surface opencode metadata (tokens, timings) if available
- Whether to add a `--json` flag for debugging raw ACP messages (currently not planned)
