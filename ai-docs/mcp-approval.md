# Theia IDE — MCP Approval Guide

Instructions for AI agents writing MCP server approval files for the Theia IDE tools.

The generic config schema is the contract: fetch `https://ai.open-vsx.org/schemas/mcp-server-config.schema.json` before writing one. The terms used here (generic config, install config, derived install config, transform, server key, notes, instructions) are defined in `ai-registry-core/CONTEXT.md`.

## Tools

The Theia organization publishes two tool ids that correspond to different distribution channels:

| Tool id | Distribution | URL scheme |
|:--------|:-------------|:-----------|
| `theia-ide` | Eclipse Theia IDE (stable) | `theia://` |
| `theia-ide-next` | Eclipse Theia IDE Next (insiders / nightly) | `theia-next://` |

**Unless an approval explicitly differs between the two channels, always add an `installConfigs` entry for both tools, identical except for the `tool` id.**

## Shape of an approval

An approval describes its server once, as a **generic config** in the root `config`, and both install configs are **derived** from it by core's Theia transform:

```json
{
    "serverId": "app.linear/linear",
    "date": "2026-06-10",
    "config": {
        "type": "sse",
        "url": "https://mcp.linear.app/sse",
        "oauth": {},
        "notes": "Remote Linear MCP server. No API key is required: on first connection your client opens a browser to authorize access to your Linear workspace via OAuth. Provides tools to manage Linear issues, projects, and teams.",
        "serverKey": "linear"
    },
    "installConfigs": [
        {
            "tool": "theia-ide",
            "config": "derived",
            "instructions": "No API key is required: on first connection Theia opens a browser to authorize access to your Linear workspace via OAuth, then stores the resulting token automatically."
        },
        {
            "tool": "theia-ide-next",
            "config": "derived",
            "instructions": "No API key is required: on first connection Theia opens a browser to authorize access to your Linear workspace via OAuth, then stores the resulting token automatically."
        }
    ]
}
```

The generic config is what every other client and the website use, so write one even when the server's own vendor publishes a generic config of their own: core prefers an organization's own config for its own install configs, so Theia keeps the connection details it reviewed.

Do not write a Theia config by hand when the transform can produce it. Leave out `installUrl` too: consolidation builds it from the tool's `mcpInstallUrlPrefix` in `organization.json`.

## The generic config

### `type`

- `stdio` for a local server (`command`, `args`, `env`).
- `sse` for a server whose endpoint speaks the legacy SSE transport (typically a URL ending in `/sse`).
- `streamable-http` for every other remote server. Never `http`: the schema accepts only the MCP spec's name.

Theia detects SSE vs Streamable HTTP itself, so `type` only matters to other clients. Get it right anyway.

### `serverKey`

The server key is the name Theia stores the server under in its MCP settings (the key under `servers`). Theia overwrites, or shows as "installed manually", a server whose key it already holds. So:

Without `serverKey`, core falls back to the `serverId`'s last segment, or, when that is only `mcp`, `server` or `mcp-server`, the nearest namespace part that says more (`com.stripe/mcp` → `stripe`). Set `serverKey` when:

- **An existing approval is converted from hand-written configs.** Set it to the key under `servers` today, even if the fallback gives the same one. Users who installed the server under the old key would otherwise get a second entry on update.
- **The fallback reads badly** for a new approval. Use a short, readable name, usually the npm package or product name (e.g. `chrome-devtools`, `brave-search`).

Otherwise leave it out. Never change a key once it has been published. Two Theia approvals with the same key fail validation.

### Placeholders: `${VAR}` vs `<name>`

- **`${VAR}`** for secrets: tokens and API keys in `env`, `headers` and `oauth.clientSecret`. Use the variable the server's own docs name (e.g. `NEON_API_KEY`). When they name none, derive one from the server and what it holds, e.g. `${ASANA_CLIENT_SECRET}`. Clients that resolve `${VAR}` (e.g. Kiro) read it from the environment or prompt for it.
- **`<name>`** for values a human has to choose and that are not secret: an instance URL, a connection string argument, a region, an OAuth `clientId`.
- **Never write a literal secret.**
- `url` must be a valid URI, so it can't hold a placeholder of either kind. A server that only takes its token in the URL can't be described as is. Use the header its docs offer instead.

Theia does **not** expand `${VAR}`. The transform rewrites `${ASANA_CLIENT_SECRET}` to `<ASANA_CLIENT_SECRET>` for Theia, which the user replaces by hand. `<name>` placeholders reach Theia unchanged.

### `notes`

Tool-neutral setup text, shown next to the generic config on the website and, through derivation, to every client:

- Explain each placeholder: what a `<name>` should be replaced with, what a `${VAR}` must hold and where to get it. Also list prerequisites (a running local service, required permissions or scopes). Leave `notes` out when there is nothing to explain.
- Name a `${VAR}` by its bare name (`EXA_API_KEY`, not `${EXA_API_KEY}`), so it reads correctly next to Theia's `<EXA_API_KEY>` too. Name a `<name>` placeholder as written.
- Keep it tool-neutral. Say "your client opens a browser", not "Theia opens a browser", and "the URL", not `serverUrl`.

## The install configs

Both entries are `"config": "derived"`.

`instructions` holds only text that is specific to Theia: Theia's own behaviour ("Theia opens a browser to authorize… then stores the resulting token automatically") or Theia's config fields (`serverAuthToken`, `serverUrl`).

- **Nothing Theia-specific to say:** leave `instructions` out. The install config then gets the generic config's notes from core.
- **Something Theia-specific to say:** write `instructions`. They **replace** the notes rather than adding to them, and Theia shows only the install config's text, never the notes. So the instructions must still explain everything a Theia user needs to finish setup, including each placeholder under the name Theia shows (`<EXA_API_KEY>`). Leave out what is only descriptive (what the server provides).

## What the transform produces

Core's Theia transform (`ai-registry-core/src/mcp-config-templates/theia.ts`) turns the generic config into Theia's MCP server preferences format. Its output is what lands in Theia's settings. Check it before you rely on it: the approval's derived config is published in `tools/theia-ide.json` after consolidation.

The output is `{ "servers": { "<serverKey>": { … } } }`.

### Local server (`stdio`)

```json
{
    "servers": {
        "<serverKey>": {
            "command": "npx",
            "args": ["-y", "<package-name>@latest"],
            "env": {
                "API_KEY": "<API_KEY>"
            }
        }
    }
}
```

`command`, `args` and `env` are copied, with `${VAR}` rewritten to `<VAR>`. Theia has no `cwd` field, so `cwd` is dropped.

For `command`, prefer `npx` for npm packages and `uvx` for Python packages. Always include `"-y"` in `args` when using `npx` to skip the install prompt, and append `@latest` to the package name.

### Remote server (`streamable-http`, `sse`)

```json
{
    "servers": {
        "<serverKey>": {
            "serverUrl": "<server-url>",
            "serverAuthToken": "<API_TOKEN>",
            "headers": { "X-Grafana-URL": "<grafana-url>" },
            "oauth": { "clientId": "<clientId>" }
        }
    }
}
```

| Theia field | Type | From the generic config |
|:------------|:-----|:------------------------|
| `serverUrl` | string | `url` |
| `serverAuthToken` | string | The auth header's value. Theia sends it as `Authorization: Bearer <token>` unless `serverAuthTokenHeader` is set. |
| `serverAuthTokenHeader` | string | The header's name, when the only header is not `Authorization` (see below). |
| `headers` | object | Every other header. |
| `oauth` | object | `oauth`, see below. |

How `headers` is split:

- An `Authorization` header becomes `serverAuthToken`, with the `Bearer ` prefix stripped because Theia adds it. Every other header goes in `headers`.
- With no `Authorization` header and exactly one other header, that header is the token: `serverAuthToken` gets its value and `serverAuthTokenHeader` its name.
- With no `Authorization` header and several others, all of them go in `headers`, since there is no way to tell which is the token.

`serverAuthToken` and `headers` therefore work together rather than as alternatives. `io.github.grafana--mcp-grafana` is an example using both.

Prefer remote servers over local (stdio) when a hosted URL is available.

### OAuth (`oauth`)

| Generic field | Theia field | Description |
|:--------------|:------------|:------------|
| `clientId` | `clientId` | Pre-registered OAuth client ID. Omit to let Theia attempt Dynamic Client Registration. |
| `clientSecret` | `clientSecret` | Client secret for confidential clients, always a `${VAR}`. Only used together with `clientId`; ignored during dynamic client registration. |
| `scopes` | `scopes` | Scopes to request, one per entry. |
| `authServerMetadataUrl` | `authorizationServer` | URL of the authorization server metadata document, overriding the default discovery chain. Note the different names. |
| `resource` | `resource` | Resource indicator (RFC 8707), for servers that require it. |

All fields are optional. Include only what the server requires. For a server that uses OAuth with dynamic client registration, write `"oauth": {}`: it tells other clients that the server expects OAuth. Theia gets no `oauth` at all, since it falls through to discovery and DCR on a 401 anyway.

### What Theia can't represent

- **WebSocket (`"type": "ws"`):** Theia has no WebSocket transport, so the transform produces no Theia config. Consolidation warns and publishes the install config without a `config`, which Theia can't install.
- **`cwd`:** dropped, as above.

## Installing from a link

Theia registers the `theia://` and `theia-next://` URL schemes with the operating system and dispatches matching links to its install handler. Consolidation sets each install config's `installUrl` to `<mcpInstallUrlPrefix><serverId>`, e.g. `theia://install-mcp?id=app.linear/linear`. Theia reads the install config, display name and version from the consolidated registry JSON at install time, so the link carries only the id, and the user installs exactly what the registry currently publishes. If the registry no longer lists the server, Theia refuses the install with a clear error.

Theia detects an update by the approval's `configHash`, a hash of the approval file's content. Any change to the content changes it. A change to core's transform alone changes the derived config but not the hash, so Theia won't offer it as an update.
