# harness-acp-bridge

Session-scoped ACP harness bridge. A single HTTP server owns harness processes, serializes
message execution, journals original traffic, and renders live output. Two independent clients
drive it over HTTP:

| Directory | Package | Role |
| --- | --- | --- |
| `server/` | `@black942026/harness-acp-bridge-server` | The HTTP server: five layers (transport / adapter / logic / HTTP / preview), twelve `POST /api/<name>` operations. Mature, unchanged. |
| `mcp/` | `@black942026/harness-acp-bridge-mcp` | Stdio MCP proxy: twelve tools (bare names) plus `wait`. Thin HTTP client; `send_message` waits for the terminal result in the same tool call and uses MCP elicitation for reverse requests when the client supports it. |
| `pi-extension/` | `@black942026/harness-acp-bridge-pi-extension` | Pi extension: twelve `acp_` tools plus `acp_wait`. Thin HTTP client with synchronous send/wait, Pi-native reverse-request dialogs, and UI-only preview. |

Each client connects **directly** over HTTP (Pi does not go through MCP). Neither client starts
a harness; both can start a **local** server on demand (loopback only) and reuse a running one.

```text
MCP host --stdio--> mcp proxy --\
                                >-- HTTP POST /api/<name> --> server --> ACP harness
Pi      --in-proc-> pi-extension --/
```

## Server

```sh
cd server && npm ci && npm run build
HARNESS_ACP_BRIDGE_CONFIG=./config.example.yaml npm start
```

`POST /api/<name>` with a JSON body; success is `{ ok: true, data }`, failure is
`{ ok: false, error: { code, message, detail } }`. Operations: `ping`, `search_harness`,
`create_session`, `close_session`, `authenticate`, `auth_status`, `set_session_options`,
`send_message`, `cancel_message`, `message_status`, `live_output`, `answer_question`. See
[`server/README.md`](server/README.md) for the authoritative contract.

## Clients

Both clients reuse a compatible server (`ping` + `protocol_version: 2`) at their configured
`baseUrl`, and start the installed server package as a detached, reused background service only
when nothing is listening on a loopback address. A non-loopback `baseUrl` is never spawned. A
reachable-but-incompatible listener is reported, never replaced. Client `shutdown`/`reload`
never closes server sessions or stops a reused instance. See each package README for lifecycle
and configuration:

- [`mcp/README.md`](mcp/README.md) — MCP tools, elicitation, `send_message`/`wait` semantics.
- [`pi-extension/README.md`](pi-extension/README.md) — `acp_` tools, preview isolation, dialogs.

Both depend on `@black942026/harness-acp-bridge-server@^0.1.2` (the HTTP five-layer server).

## Migration from the old socket/MCP wiring

The server is HTTP-only; the old Unix-socket daemon, the shared `harness_info`/`auth_info`/
`set_model`/`message_result` names, `mode`/`queue`/`steering` parameters, and the
`offset`/rolling-window preview protocol are gone. The clients now speak HTTP directly:

- Pi settings no longer take `command`/`args`/`env`/`cwd`; use `baseUrl` (and optionally
  `autoStart`/`serverEntry`/`configPath`). Unknown fields are rejected.
- The MCP proxy is configured by environment/JSON (`HARNESS_ACP_BRIDGE_URL`, ...), not a
  spawned CLI.
- `live_output` uses `cursor`/`limit`/`wait_ms`; `answer_question` takes a string `request_id`.

## Development

Each package is an independent git repository. `cd` into one and run `npm ci && npm run check`;
the client checks start the real server and its fake ACP harness for end-to-end coverage (no paid
model, Docker, or SSH).

## License

MIT
