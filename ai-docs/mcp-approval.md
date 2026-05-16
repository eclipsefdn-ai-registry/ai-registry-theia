# Theia IDE — MCP Approval Config Guide

Instructions for AI agents generating MCP server approval files for the Theia IDE tool (`theia-ide`).

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
- For `command`, prefer `npx` for npm packages, `uvx` for Python packages.
- Always include `"-y"` in `args` when using `npx` to skip the install prompt.
- Append `@latest` to the package name in `args` to ensure the latest version.
- Use `<placeholder>` syntax for any secrets or tokens so the user knows to replace them.
- Add an `instructions` field to the `installConfig` explaining what the user needs to configure.
