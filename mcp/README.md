# CuePool MCP server

This package is a **STDIO MCP sidecar** for a running CuePool instance. It
proxies MCP tool calls to CuePool's `/v1` automation API and does not contain
show-control logic of its own.

See also: [Automation API reference](../docs/AUTOMATION.md)

## Safety and operator consent

Control tools can trigger live playback and external outputs (audio, video,
lighting, DMX/network integrations). Treat them like pressing show controls on
the operator desk:

- only enable control access with explicit operator consent,
- use read-only mode by default when possible,
- follow Leave No Trace practices (do not leave surprise armed automations or
  unattended control sessions connected to a live rig).

## Prerequisites

1. **CuePool is running** with the automation API reachable at
   `http://127.0.0.1:7133/v1` (default bind).
2. **Node.js 20+** is installed.
3. For control tools, CuePool must be launched with
   `CUEPOOL_API_CONTROL_TOKEN` set.

Quick readiness check:

```sh
curl http://127.0.0.1:7133/v1/health
```

## Build

```sh
cd mcp
npm ci
npm run build
```

The server entrypoint is `mcp/dist/index.js`.

## Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `CUEPOOL_API_URL` | No | Automation API origin. Defaults to `http://127.0.0.1:7133`. Must be an HTTP(S) origin only (no path/query/fragment). Plain `http://` is accepted only for loopback hosts (`127.x.x.x`, `localhost`, `::1`); use `https://` for remote/tunneled targets. |
| `CUEPOOL_API_TOKEN` | No | Bearer token used for control calls (`POST /v1/commands`). Set this to the same value as CuePool's `CUEPOOL_API_CONTROL_TOKEN` when you want control tools enabled. |
| `CUEPOOL_API_TIMEOUT_MS` | No | Request/command timeout in milliseconds. Default `10000`. Valid range: integer `100..300000`. |

## Read-only vs control mode

- **No `CUEPOOL_API_TOKEN`**: only read tools are advertised:
  `cuepool_health`, `cuepool_project`, `cuepool_cues`, `cuepool_active_cues`,
  `cuepool_diagnostics`, `cuepool_logs`.
- **With `CUEPOOL_API_TOKEN`**: the server also advertises control tools
  (`cuepool_select_cue`, `cuepool_go`, `cuepool_stop`, `cuepool_pause`,
  `cuepool_resume`, `cuepool_preload`, `cuepool_seek`, `cuepool_shutdown`).

Authentication is not bypassed: if CuePool was started without
`CUEPOOL_API_CONTROL_TOKEN`, command calls still fail with
`403 control_disabled` even if this sidecar has `CUEPOOL_API_TOKEN` set.

## `operation_id` and idempotency

Every control tool requires an `operation_id`. The sidecar forwards it as the
`Idempotency-Key` header to CuePool.

- Use a unique `operation_id` for each new command attempt.
- If a result is uncertain (transport interruption, timeout, etc.), retry with
  the **same** `operation_id` so CuePool can return the original command result
  instead of executing the command twice.

## Grok Bot / Cursor MCP wiring (local loopback)

Use a custom **stdio** MCP server entry in Cursor/Grok Bot and point it at the
built `dist/index.js` with an absolute path.

1. Copy `mcp/grok-bot.mcp.example.json`.
2. Replace `/absolute/path/to/cuePool` with your checkout path.
3. Keep or remove `CUEPOOL_API_TOKEN` depending on whether you want read-only
   or control mode.

Minimal example:

```json
{
  "mcpServers": {
    "cuepool": {
      "command": "node",
      "args": [
        "/absolute/path/to/cuePool/mcp/dist/index.js"
      ],
      "env": {
        "CUEPOOL_API_URL": "http://127.0.0.1:7133"
      }
    }
  }
}
```

Control-enabled example:

```json
{
  "mcpServers": {
    "cuepool-control": {
      "command": "node",
      "args": [
        "/absolute/path/to/cuePool/mcp/dist/index.js"
      ],
      "env": {
        "CUEPOOL_API_URL": "http://127.0.0.1:7133",
        "CUEPOOL_API_TOKEN": "replace-with-your-cuepool-control-token"
      }
    }
  }
}
```

## Notes

- `cuepool_shutdown` targets only the CuePool profile serving
  `CUEPOOL_API_URL`, returns CuePool's final acknowledgement, and is rejected
  while playback is active or when unsaved changes exist.
- This MCP process is a sidecar only. It does **not** launch CuePool.
