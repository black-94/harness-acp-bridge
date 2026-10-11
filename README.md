# harness-acp-bridge

Session-scoped ACP harness bridge. A single HTTP server owns harness processes, serializes
message execution, and journals original traffic. Two independent clients drive it over HTTP:

| Directory | Package | Role |
| --- | --- | --- |
| `server/` | `@black942026/harness-acp-bridge-server` | The HTTP server: eleven `POST /api/<name>` operations (transport / adapter / logic / HTTP layers). |
| `mcp/` | `@black942026/harness-acp-bridge-mcp` | Stdio MCP proxy: eleven tools (bare names) plus `wait`. Thin HTTP client; `send_message` waits for the terminal result in the same tool call and uses MCP elicitation for reverse requests when the client supports it. |
| `pi-extension/` | `@black942026/harness-acp-bridge-pi-extension` | Pi extension: eleven `acp_` tools plus `acp_wait`. Thin HTTP client with synchronous send/wait, Pi-native reverse-request dialogs, and UI-only progress preview. |

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
`send_message`, `cancel_message`, `message_status`, `answer_question`. See
[`server/README.md`](server/README.md) for the authoritative contract.

The server advertises `protocol_version: 3`. Authenticate is a server-owned, interactive flow:
`authenticate` returns a flow snapshot, the client asks each returned question and submits answers
with a session-scoped `answer_question` (no `message_id`), and `auth_status` polls the same flow.
A login-gated `create_session`/`send_message` returns a structured `authentication_required` error
whose `detail` carries `session_id` and `next_action`; the client never auto-authenticates.

## Clients

Both clients reuse a compatible server (`ping` + `protocol_version: 3`) at their configured
`baseUrl`, and start the server as a detached, reused background service only when nothing is
listening on a loopback address. A non-loopback `baseUrl` is never spawned. A
reachable-but-incompatible listener — including an older-protocol (e.g. `protocol_version: 2`)
daemon — is reported and **rejected**, never replaced, restarted, or killed. Client
`shutdown`/`reload` never closes server sessions or stops a reused instance. See each package
README for lifecycle and configuration:

- [`mcp/README.md`](mcp/README.md) — MCP tools, elicitation, authentication, `send_message`/`wait`.
- [`pi-extension/README.md`](pi-extension/README.md) — `acp_` tools, preview isolation, dialogs, auth.

Both depend on `@black942026/harness-acp-bridge-server` (the HTTP server, protocol 3). There is no
separate incremental-output endpoint: clients derive their UI-only preview from the optional bounded
`status.progress` snapshot and use the terminal `result` as the final answer.

## Migration

The server is HTTP-only; the old Unix-socket daemon, the shared `harness_info`/`auth_info`/
`set_model`/`message_result` names, and `mode`/`queue`/`steering` parameters are gone. This release
also makes a breaking contract change:

- The `live_output` operation and its `acp_live_output` / `live_output` tools are **removed** (no
  replacement endpoint). Progress is UI-only via `message_status.progress`; the final answer is
  `message_status.result`.
- Authentication is a server-owned flow. `authenticate` no longer takes a `method_id` (the method
  is selected from the returned candidates through the client's own UI) and `create_session` no
  longer takes an `authenticate` parameter. `answer_question`'s `message_id` is optional — omitted
  for session-authentication questions and reused for the shared `questions` shape
  (`request_id`/`kind`/`title`/`message`/`options`/`schema`).
- The client compatibility protocol is now 3; older daemons are refused rather than reused.

Client configuration:
- Pi settings no longer take `command`/`args`/`env`/`cwd`; use `baseUrl` (and optionally
  `autoStart`/`serverEntry`/`configPath`). Unknown fields are rejected.
- The MCP proxy is configured by environment/JSON (`HARNESS_ACP_BRIDGE_URL`, ...), not a
  spawned CLI.

## Development

Each package is an independent git repository. `cd` into one and run `npm ci && npm run check`.
Client integration/lifecycle suites start a real server and its fake ACP harness (no paid model,
Docker, or SSH); they resolve the in-repo build (`../server/dist`) first and fall back to the
installed package, so they run only against a protocol-3 server and are skipped until the server is
rebuilt with the new contract.

## License

MIT
