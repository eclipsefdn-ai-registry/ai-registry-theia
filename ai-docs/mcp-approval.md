# Theia IDE — MCP Approval Config Guide

Instructions for AI agents generating MCP server approval files for the Theia IDE tools.

## Tools

The Theia organization publishes two tool ids that correspond to different distribution channels:

| Tool id | Distribution | URL scheme |
|:--------|:-------------|:-----------|
| `theia-ide` | Eclipse Theia IDE (stable) | `theia://` |
| `theia-ide-next` | Eclipse Theia IDE Next (insiders / nightly) | `theia-next://` |

**Unless an approval explicitly differs between the two channels, always add an `installConfig` entry for both tools with the same `config` and `instructions`. Only the `tool` id and the `installUrl` scheme change between the two entries.** This keeps coverage symmetrical so both distributions surface the same approved servers.

## Config Format

The `config` object in an `installConfig` entry must follow the Theia MCP server preferences schema. The top-level key is `servers`, containing one entry per server keyed by a short name.

Each server entry supports two modes: **local** (stdio) and **remote** (SSE/HTTP).

### Local server (command-based)

```json
{
  "servers": {
    "<server-name>": {
      "command": "npx",
      "args": ["-y", "<package-name>@latest"],
      "env": {
        "API_KEY": "<placeholder>"
      }
    }
  }
}
```

| Field | Type | Description |
|:------|:-----|:------------|
| `command` | string | Command to execute (e.g., `npx`, `uvx`, `node`) |
| `args` | string[] | Arguments passed to the command |
| `env` | object | Optional environment variables (e.g., API keys). Use `<placeholder>` values for secrets. |

### Remote server (URL-based)

```json
{
  "servers": {
    "<server-name>": {
      "serverUrl": "<server-url>",
      "serverAuthToken": "<placeholder>",
      "serverAuthTokenHeader": "Authorization"
    }
  }
}
```

| Field | Type | Description |
|:------|:-----|:------------|
| `serverUrl` | string | URL of the remote MCP server |
| `serverAuthToken` | string | Authentication token. Use `<placeholder>` value. |
| `serverAuthTokenHeader` | string | Optional. Header name for the token. Defaults to `Authorization` with `Bearer` prefix if omitted. |
| `headers` | object | Optional additional headers included with each request. |

### Fields to omit

- `autostart` — Do not include. This is a user preference, not part of the approval.
- `name` — Do not include. The server key serves as the identifier.

## Guidelines

- Use the npm package name from the Anthropic MCP registry as the `<server-name>` key (e.g., `chrome-devtools`, `brave-search`).
- Prefer remote servers over local (stdio) when a hosted/cloud URL is available.
- For `command`, prefer `npx` for npm packages, `uvx` for Python packages.
- Always include `"-y"` in `args` when using `npx` to skip the install prompt.
- Append `@latest` to the package name in `args` to ensure the latest version.
- Use `<placeholder>` syntax for any secrets or tokens so the user knows to replace them.
- Add an `instructions` field to the `installConfig` explaining what the user needs to configure.

## installUrl (one-click install for Theia)

Always add an `installUrl` field to the `installConfig` so users can install the server with a single click from a website. Theia IDE registers the `theia://` URL scheme with the operating system and dispatches matching links to its install handler.

### Format

```
<scheme>://install-mcp?id=<serverId>
```

The scheme matches the tool: `theia://` for `theia-ide`, `theia-next://` for `theia-ide-next`. The URL carries only the approval's `serverId`. Theia reads the corresponding install configuration, display name, and version directly from the consolidated registry JSON at install time — there is no encoded payload. This keeps install links short and ensures the user installs exactly what the registry currently publishes.

### Constructing the URLs

```js
const id = encodeURIComponent(approval.serverId);
const stableUrl = `theia://install-mcp?id=${id}`;       // for tool: theia-ide
const nextUrl   = `theia-next://install-mcp?id=${id}`;  // for tool: theia-ide-next
```

### Rules

- The `id` query parameter **must** match the approval's top-level `serverId`.
- The URL scheme **must** match the `tool` id: `theia://` for `theia-ide`, `theia-next://` for `theia-ide-next`.
- Do not embed `config`, `localSlug`, `name`, or `version` in the URL — Theia fetches all of that from the registry by ID.
- The Theia client may refuse to install if the registry does not list the given `serverId` (e.g., the registry has been updated to remove the server). The link will still surface a clear error message to the user.
