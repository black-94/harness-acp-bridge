# harness-acp-bridge

Session-scoped ACP harness bridge. A single HTTP server owns harness processes, serializes
message execution, and journals original traffic. Two independent clients drive it over HTTP:

| Directory | Package | Role |
| --- | --- | --- |
| `server/` | `@black942026/harness-acp-bridge-server` | The HTTP server: fourteen `POST /api/<name>` operations (transport / adapter / logic / HTTP layers), including `register_template` / `search_template` / `delete_template` (templates are stored **information only**) and a template-free `create_session`. |
| `mcp/` | `@black942026/harness-acp-bridge-mcp` | Stdio MCP proxy: thirteen tools (bare names) plus `wait`. Thin HTTP client; `send_message` waits for the terminal result in the same tool call and uses MCP elicitation for reverse requests when the client supports it. `search_template`/`create_template` expose the server's templates (`create_template` is a client alias for `register_template`). |
| `pi-extension/` | `@black942026/harness-acp-bridge-pi-extension` | Pi extension: thirteen `acp_` tools plus `acp_wait`. Thin HTTP client with synchronous send/wait, Pi-native reverse-request dialogs, and UI-only progress preview. `acp_search_template`/`acp_create_template` expose the server's templates, and a portable `acp-create-template` skill guides collection. |

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
`{ ok: false, error: { code, message, detail } }`. Operations: `ping`,
`register_template`, `delete_template`, `search_template`, `create_session`, `close_session`,
`authenticate`, `auth_status`, `set_session_options`,
`send_message`, `cancel_message`, `message_status`, `answer_question`. See
[`server/README.md`](server/README.md) for the authoritative contract.

The server advertises `protocol_version: 4`. Authenticate is a server-owned, interactive flow:
`authenticate` returns a flow snapshot, the client asks each returned question and submits answers
with a session-scoped `answer_question` (no `message_id`), and `auth_status` polls the same flow.
A login-gated `create_session`/`send_message` returns a structured `authentication_required` error
whose `detail` carries `session_id` and `next_action`; the client never auto-authenticates.

## Clients

Both clients reuse a compatible server (`ping` + `protocol_version: 4`) at their configured
`baseUrl`, and start the server as a detached, reused background service only when nothing is
listening on a loopback address. A non-loopback `baseUrl` is never spawned. A
reachable-but-incompatible listener — including an older-protocol (e.g. `protocol_version: 3`)
daemon — is reported and **rejected**, never replaced, restarted, or killed. Client
`shutdown`/`reload` never closes server sessions or stops a reused instance. See each package
README for lifecycle and configuration:

- [`mcp/README.md`](mcp/README.md) — MCP tools, elicitation, authentication, `send_message`/`wait`.
- [`pi-extension/README.md`](pi-extension/README.md) — `acp_` tools, preview isolation, dialogs, auth.

Both depend on `@black942026/harness-acp-bridge-server` (the HTTP server, protocol 4). There is no
separate incremental-output endpoint: clients derive their UI-only preview from the optional bounded
`status.progress` snapshot and use the terminal `result` as the final answer.

## Launch templates

Launch templates are administrator-owned **information**, persisted by the server so a caller can
read a known-good recipe — they are never a `create_session` parameter. `create_session` takes the
launch executable directly (`command`/`args`/`env`, plus Docker-only `login_dirs`) along with
`cwd`/`model_id` and the other session fields; a `template`/`name`/`description` key is rejected.
Both clients expose the same workflow:

1. The user names or describes a template. `search_template` (MCP) / `acp_search_template` (Pi)
   finds it by exact `name`, or `query`/`harness`; if several match, ask the user which one. The
   full template content is returned so the model can expand it.
2. Read the object, map non-null `model` → `model_id` and `thinking` → `thinking_level`, and apply
   user overrides. Ask only for a missing model or `cwd`; no static YAML candidate lists are needed.
3. Follow `special_instructions` throughout. The calling LLM executes natural-language
   `before_start` actions before startup and `after_start` actions after successful startup and any
   required authentication, subject to user approval and safety rules. Registration executes nothing;
   failed/unavailable actions must be reported, not silently skipped or claimed successful.
4. Map Docker `network_mode` to session `host_network` (`host` → true; `none`/`proxy` → false),
   pass the returned `env`, and **drop** `name`/`description`/`model`/`thinking`/`network_mode`/
   `proxy_port`/`before_start`/`after_start`/`special_instructions`
   and inapplicable fields, then call `create_session(cwd, model_id, …)` with only explicit allowed
   fields. Omit invalid nulls (local `remote_host`, missing `cwd`) and re-check combinations after
   runtime/target overrides. Neither server nor clients execute hooks or automatically merge a
   template; the session snapshot carries no template identity.

Register a template with `create_template` (MCP) / `acp_create_template` (Pi) — a client alias for
the server's `register_template` (there is no `POST /api/create_template`); duplicates are rejected,
never overwritten, and registration starts no harness. The Pi package ships an
[`acp-create-template` skill](pi-extension/skills/acp-create-template/SKILL.md) that guides the
interview; it is a portable Agent Skills directory, so an MCP host can load the same `SKILL.md`.

Docker templates use `network_mode: none | host | proxy`. Proxy mode requires only `proxy_port`
(1–65535); registration generates identical lowercase `http_proxy`/`https_proxy` using
`http://host.docker.internal:PORT` and a `no_proxy` list for localhost/private ranges, replacing stale
proxy aliases while preserving other env. It is not tied to Xray. Old template `host_network` is
rejected; re-register templates using the new fields. See the server README for reachability and
CIDR-client support requirements; registration does not test a live proxy.

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
- `create_session` is decoupled from templates: it no longer accepts `template`/`name`/`description`,
  never looks up or merges a template, and never records a selected-template identity. Templates are
  stored **information only** (still `register_template`/`search_template`/`delete_template`).
  `create_session` now accepts the launch executable directly (`command`/`args`/`env`) and
  Docker-only `login_dirs`; `cwd` and `model_id` are always required. A caller chooses which
  executable launches; the ACP handshake validates the harness name when `agentInfo` is supplied.
  ACP v1 permits omitted/null `agentInfo`, which does not block creation.
- The client compatibility protocol is now 4 (was 3); older daemons — including a running
  protocol-3 server, which must be restarted by its operator — are refused rather than reused,
  restarted, or killed.

`search_harness` and its Pi/MCP tools are removed; use launch templates to select a harness and
model/thinking defaults.

Configuration migration: remove the entire obsolete `harnesses` section from server YAML.
All three built-in adapters are always available; select one from a template or explicit session
`harness`. Omitted harness uses `default_harness` (Codex by default). With all service settings
at defaults, no YAML file is needed. Use template `model`/`thinking` defaults or explicit session
options; validation comes from the live harness. Existing templates load with null values for all
five new fields.

Client configuration:
- Pi settings no longer take `command`/`args`/`env`/`cwd`; use `baseUrl` (and optionally
  `autoStart`/`serverEntry`/`configPath`). Unknown fields are rejected.
- The MCP proxy is configured by environment/JSON (`HARNESS_ACP_BRIDGE_URL`, ...), not a
  spawned CLI.

## Development

Each package is an independent git repository. `cd` into one and run `npm ci && npm run check`.
Client integration/lifecycle suites start a real server and its fake ACP harness (no paid model,
Docker, or SSH); they resolve the in-repo build (`../server/dist`) first and fall back to the
installed package, so they run only against a protocol-4 server and are skipped until the server is
rebuilt with the new contract.

## License

MIT
