# mspbots-rmm-mcp

MCP server for the **MSPbots RMM Control API** (`mb-platform-rmm`) — a
vendor-agnostic remote monitoring and management console: device
inventory, script management, and remote execution, proxied through a
vendor adapter so the same endpoints work whichever RMM a tenant has
connected.

It follows the same design as the sibling `mspbots-agent-mcp` service:
stateless, no stored credentials, per-request header authentication over the
[Model Context Protocol](https://modelcontextprotocol.io/) (Streamable HTTP/SSE
transport).

> **Renamed from `mspbots-fleet-mcp`.** The backend product was renamed
> `mb-platform-fleet` → `mb-platform-rmm` to reflect that it now proxies
> multiple RMM vendors, not just fleetdm. This is a breaking change, not a
> relabeling — see **Migration notes** below.

<div align="center">

## MSPbots Connector

**The MCP Gateway built for MSPs: visible, controlled, and your data stays where it belongs.**

[**Get started at connect.mspbots.ai →**](https://connect.mspbots.ai)

</div>

---

MSPbots Connector gives your AI clients one MCP URL to reach your entire MSP stack (RMM, PSA, documentation, security, finance), and gives you control over who can call which tool and a record of everything that was called.

**Hosted service: [connect.mspbots.ai](https://connect.mspbots.ai)** · Free to use.

### Why Connector

![MSPbots Connector](https://stgoss.mspbots.ai/files/mcp/mspbots-mcp.png)

| | |
|---|---|
| **Visible** | Every tool call is logged: who, which vendor, when. Usage dashboard, CSV export, SIEM streaming. |
| **Controlled** | Role-based, per-tool allowlist. Deny by default. |
| **Protected** | Credentials encrypted per record. AI clients never see your vendor passwords or tokens. |
| **MSP-native** | Built for the MSP stack only: RMM, PSA, documentation, security, finance. |

### Quick start

1. **Sign up** at [connect.mspbots.ai](https://connect.mspbots.ai).
2. **Connect your vendors.** Pick from 150+ connectors and authorize each one on the vendor's own page, or bring your own API keys. No extra vendor accounts needed.
3. **Set access.** Create roles and decide which tools each role may call. Nothing is allowed until you allow it.
4. **Add your MCP URL to your AI client.** Copy the URL from your Connector dashboard into Claude, ChatGPT, Cursor, VS Code, Gemini, Codex, or any MCP-capable client.
5. **Ask your AI to do real work**, and watch every call appear in the audit log.

### Features

**Unified gateway**
- One MCP URL for all your AI clients
- 150+ connectors across the MSP stack
- Custom MCP servers supported, free
- Authorization happens on each vendor's own page; the AI never sees passwords or tokens
- Bring your own credentials: use the vendor API keys you already have
- Multiple accounts per app; connect once, reuse across AI clients
- Tool search: finds the right action for the task
- One sign-in; vendor credentials are injected behind the scenes and OAuth tokens refresh automatically

**Visibility**
- Audit log for every tool call (who, which vendor, when)
- Usage dashboard: call volume, top vendors, error rate, per-user activity
- Export audit logs to CSV, or stream to Loki, Graylog, or LogScale (SIEM)
- Optional recording of tool arguments and result summaries (off by default, enabled per organization)

**Control**
- Per-role, per-tool allowlist, deny by default
- owner / admin / member roles
- SCIM provisioning with Entra ID, Okta, JumpCloud, and Google Workspace

**Security**
- Credentials encrypted one record at a time (AES-256-GCM, per-record key)
- AI clients authenticate with OAuth 2.1 + PKCE
- Vendor responses are scanned for prompt-injection patterns and returned as structured data. This is one layer of defense, not a guarantee against every attack.
- Static egress IP for customer firewall and API allowlisting

### Pricing

**Free.**

Links: [connect.mspbots.ai](https://connect.mspbots.ai) · [mspbots.ai](https://mspbots.ai)

---

## When would you use this

This MCP wraps an internal MSPbots product (device/RMM management), not a
third-party integration. Typical uses:

- "What devices does this tenant have, and which are offline?" →
  `mspbotsrmm_list_devices`
- "This machine looks stale, force a refresh" → `mspbotsrmm_refetch_device`
- "Run our log-collection script on that server" →
  `mspbotsrmm_run_script` (dry-run first, then confirm)
- "Did that script run finish? What was the output?" →
  `mspbotsrmm_list_script_runs`

## Tools

Credentials for all tools come from request headers (`X-API-Key` / `X-MSP-Tenant-Id` / `X-MSP-Host`),
so no token needs to be passed in tool parameters. In addition to the parameters listed below, **every tool also accepts an optional
`connection`** (RMM connection UUID); it can be omitted when the tenant has connected only one RMM, and
must be passed explicitly only when several are connected.

### Devices (read-only)

Devices cannot be created or deleted through the API: a machine appears automatically once it has the connected RMM client installed and registered, and disappears automatically after it is removed
upstream.

| Tool | Description | Parameters |
|---|---|---|
| `mspbotsrmm_list_devices` | List the current tenant's devices (fuzzy search + status/platform filters + pagination) | `search`, `status`(`online`/`offline`/`missing`), `platform`, `page`(default 0), `per_page`(default 25, max 200) |
| `mspbotsrmm_get_device` | Get device details: hardware identifiers (`identifiers`), vendor-specific attributes (`attributes`), and the list of capabilities supported by the connection (`capabilities`) | `id`(required) |
| `mspbotsrmm_refetch_device` | Ask the device to refresh its attributes at its next check-in (asynchronous; not supported by all vendors) | `id`(required) |

### Scripts

Scripts are stored in the connected RMM, not locally in this application: every write is passed straight through with no local copy, so changes
are immediately visible to people using that RMM directly. **Some vendors' script libraries are read-only**, and create/update/delete
return a permanent error when unsupported (see "Capability gating" below).

| Tool | Description | Parameters |
|---|---|---|
| `mspbotsrmm_list_scripts` | List this tenant's script library, with the most recent run result for each script | `page`, `per_page` |
| `mspbotsrmm_get_script` | Get script details (including the body) | `id`(required) |
| `mspbotsrmm_create_script` | Create a script. The file name extension determines the interpreter and is validated per vendor rules (for example Fleet: `.sh` may have an optional shell shebang, `.py` must have `#!/usr/bin/env python3`, `.ps1` must not have a shebang). **Not idempotent**: repeated calls create multiple scripts | `name`(required), `contents`(required) |
| `mspbotsrmm_update_script` | Replace the whole script name + body (not a partial patch; the full content must be passed) | `id`(required), `name`(required), `contents`(required) |
| `mspbotsrmm_delete_script` | Permanently delete a script (takes effect for all devices; cannot be undone) | `id`(required) |

### Execution & history

Execution is asynchronous and pull-based: the RMM hands the task to the client to run at its next check-in, and the result is not in the dispatch response;
poll `list_script_runs` to get it.

| Tool | Description | Parameters |
|---|---|---|
| `mspbotsrmm_run_script` | Run a script on one or more devices. **`confirm` defaults to `false` = dry-run preview only, nothing is actually dispatched**; only `confirm=true` really executes. After dispatch it returns one commandId per device immediately, without output/exit code | `id`(required), `device_ids`(required, at least 1), `confirm`(default false), `parameters`(optional) |
| `mspbotsrmm_list_script_runs` | Read a script's run history (newest first), including exit code, output, and `matchConfidence`(`exact`\|`heuristic`) | `id`(required), `page`(default 0), `per_page`(default 20, max 200) |

> `run_script` dispatches to devices one by one: if a device is rejected midway, the earlier devices are already running,
> and the call still fails; compare the number of returned commandIds with the number of `device_ids` to tell whether
> the dispatch partially succeeded.
>
> Run states: `queued`/`dispatched`/`running`(non-terminal) → `succeeded`/`failed`/
> `completed_unknown`/`expired`(terminal). `completed_unknown` means the RMM confirmed completion but did not
> return an exit code; `expired` means no result was received within 15 minutes (the script may actually have run and only the receipt was lost).

### Capability gating

Not every RMM vendor supports every operation. The `capabilities`
array returned by `mspbotsrmm_get_device` declares which capabilities the connection actually supports. Calling an operation the current connection does not support returns a **permanent,
non-retryable** error (HTTP 501, with `code` set to `not_supported` in the error envelope), and the message
names the specific vendor, instead of failing silently or returning an empty result.

Always available: `list_devices`, `get_device`, `list_scripts`, `get_script`,
`run_script`. May vary by vendor (501): `refetch_device`, `create_script`,
`update_script`, `delete_script`; `list_script_runs` still works when the corresponding capability is missing,
it just skips the live check against upstream.

> **`reboot_device` is not registered as a tool**: the backend route exists, but so far no vendor adapter
> implements it, so every call returns 501 (Fleet does not even have a reboot API). It will be added once an adapter
> actually implements it.

## Migration notes (migrating from `mspbots-fleet-mcp`)

The backend `mb-platform-fleet` has been renamed to `mb-platform-rmm` with incompatible API changes, so this repository
was renamed as a whole (repository/package name/tool prefix `mspbotsfleet_*` → `mspbotsrmm_*`). **The old
`mspbotsfleet_*` tools will disappear from the gateway**, and agent configurations that reference them must be updated accordingly. Main changes:

- Base path: `/apps/mb-platform-fleet/api/fleet` → `/apps/mb-platform-rmm/api/rmm`
- **Resource model**: `hosts` → `devices`, adding the `connectionId`/`identifiers[]`/`attributes{}`/`capabilities[]` fields
- **The Saved queries (osquery) concept is removed entirely**: `mspbotsfleet_*_query` (6 tools) have no equivalent in the new API, and the vendor-agnostic abstraction layer no longer supports osquery
- **Host claim/release (claiming mechanism) is removed entirely**: `mspbotsfleet_claim_hosts`/`release_host`/`set_host_labels`/`delete_host`/`list_host_scripts`(5 tools) and the `X-Claim-Hosts` header logic at connection time are all gone; devices are managed entirely by the vendor's own join/leave mechanism
- `run_script` semantics changed: old single `host_id` + `sync` flag (synchronous wait) → new `device_ids[]` batch dispatch + `confirm` flag (dry-run preview / real execution), now asynchronous and pull-based
- `get_script_result` (single lookup by execution_id) → `list_script_runs` (history list by script id, with `matchConfidence` marking how reliable the result match is)
- Added `refetch_device`; the `reboot_device` route exists but is not registered (see the Capability gating notes above)
- Upstream error envelope format: `{message, code}` → `{message, detail, code}`(the `{error:{code,message,retryable}}` envelope this service exposes is unchanged; it just also reads `detail` internally), and adds the 501 status code (capability gating)

## Quick Start

### Docker (recommended)

```bash
docker compose up --build
```

The server starts on `http://localhost:8080`.

### Local (uv)

```bash
uv sync
python -m mspbots_rmm_mcp
```

## Health Check

```bash
curl http://localhost:8080/health
# {"status": "ok"}
```

No credentials are required for the health endpoint.

## Authentication

Every request to `/mcp` must include the following HTTP headers (provided by the
MCP caller — kept consistent with `mspbots-agent-mcp` / `ticketqa-mcp`):

| Header | Type | Required | Description | Example |
|---|---|---|---|---|
| `X-API-Key` | string | Required | Access credential issued by the RMM Control API (JWT bearer token). This service forwards it unchanged as the downstream request's `Authorization: Bearer <token>`. | `X-API-Key: <jwt-bearer-token>` |
| `X-MSP-Tenant-Id` | string | Required | Tenant identifier. Renamed to the `X_Tenant_ID` header when forwarded to the downstream API (the tenant is also embedded in the JWT). | `X-MSP-Tenant-Id: <tenant-id>` |
| `X-MSP-Host` | string | Required | The host where the RMM Control API runs. | `X-MSP-Host: https://agent.mspbots.ai` |

> ⏳ **Transition-period compatibility (from 2026-09-23)**: `X-API-Key` replaces the original `X-MSP-Token`. Tenant credentials written before the rename
> still exist in the registry under the old name and are injected unchanged by the gateway, so the middleware falls back to reading
> `X-MSP-Token` when `X-API-Key` is absent; when both are present, `X-API-Key` wins. Once all tenant credentials have been re-saved under the new name, delete
> that fallback in `server.py` and `tests/test_middleware.py::test_legacy_token_header_is_still_accepted`.
>
> `X-MSP-Tenant-Id` is **not affected by this rename**; it is still required and still forwarded downstream.

Missing any of the three required headers returns `401 Unauthorized`.

> The downstream RMM Control API itself returns
> **403** for all authentication failures (missing token / bad signature / insufficient role), not 401; 401 only occurs at this service's own layer (missing gateway header). Unsupported capability
> (the vendor did not implement the feature) returns **501**, a permanent error; do not retry.

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MCP_HTTP_PORT` | `8080` | Listening port |
| `MCP_HTTP_HOST` | `0.0.0.0` | Listening host |

## MCP Endpoint

```
POST http://localhost:8080/mcp
```

Connect your MCP client with:
- Transport: `http` (Streamable HTTP / SSE)
- Headers: `X-API-Key`, `X-MSP-Tenant-Id`, `X-MSP-Host` (all required)

## Test Example

```bash
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "X-API-Key: <token>" \
  -H "X-MSP-Tenant-Id: <tenant-id>" \
  -H "X-MSP-Host: https://agent.mspbots.ai" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": { "name": "mspbotsrmm_list_devices", "arguments": { "status": "online", "per_page": 10 } }
  }'
```

> ⚠️ This repository is public. Do not write real tokens / tenant ids or other sensitive information into any committed file;
> the `<token>` / `<tenant-id>` above are placeholders only.

## Known Gaps

- **Not yet live-verified against the new `mb-platform-rmm` backend.** This
  rewrite was built entirely from the interface spec provided for the
  rename (route handlers `service/routes/scripts.ts` /
  `service/routes/devices.ts`), matching the same discipline the previous
  `mspbots-fleet-mcp` build followed for its own API version — schema/tool
  count confirmed via the real MCP protocol (`tools/list`, 10 tools) and 18
  unit tests passing, but no call has yet been exercised against a real
  tenant/token on the new backend.
- **Capability gating is enforced server-side only, not at tool-registration
  time.** The upstream doc suggests registering only the tools a
  connection's live capability list supports — structurally not possible
  here, since tool registration happens once at process startup (before
  any tenant's credentials exist), while capabilities are per-connection
  and only knowable per-request. All 10 tools are always registered; an
  unsupported call surfaces as the same 501 `not_supported` error envelope
  every other error goes through, naming the vendor.
- **`GET /vendors`** (per-vendor script-extension/naming rules referenced in
  `mspbotsrmm_create_script`'s description) **and `GET /connections`** (raw
  per-connection capability list) are reference endpoints only, not
  exposed as MCP tools — the doc does not tag either with a tool name.
