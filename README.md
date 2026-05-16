# mcproxy

Aggregate multiple upstream MCP servers (stdio, SSE, streamable-HTTP)
into a single logical MCP exposed on one HTTP endpoint. Tools, prompts,
resources, and resource templates from each upstream are prefixed with
the upstream's name so they don't collide. Credentials are resolved via
pluggable providers using `${type:name}` template syntax in any
templated string field.

## Running

```
mcproxy -config mcproxy.json
```

## Config

Top-level shape:

```json
{
  "proxy":     { "addr": ":8080", "path": "/", "transport": "streamable" },
  "providers": [ ... ],
  "servers":   { ... }
}
```

- `proxy.addr`: listen address.
- `proxy.path`: mount path for the MCP HTTP handler. Defaults to `/`.
- `proxy.transport`: `streamable` (default) or `sse`.

### Templated strings

Any string field in a `servers` entry (`command`, `args`, `env` values,
`url`, `headers`) may contain `${type:name}` references. `type` matches
a provider's `type`; `name` is a value claimed by that provider. A
field can mix literals and references:

```
"Authorization": "Bearer ${env:GITHUB_TOKEN}"
```

The proxy refuses to start if a reference points at no provider.
Provider config fields are **not** templated.

## Providers

### `env`: process environment

```json
{ "type": "env" }
```

Every environment variable is claimed by `env`. Reference as
`${env:VAR_NAME}`. Missing variables error at resolve time.

### `sops`: SOPS-encrypted JSON file

```json
{
  "type": "sops",
  "path": "/etc/mcproxy/secrets.enc.json",
  "bin": "sops"
}
```

- `path`: path to the encrypted file (required).
- `bin`: sops binary to shell out to. Defaults to `sops`.

The file is decrypted once at startup via `sops decrypt --output-type
json <path>`. Top-level keys become claimed names. Reference as
`${sops:my_key}`. Values must be JSON strings.

### `oauth`: OAuth 2.0 client credentials

```json
{
  "type": "oauth",
  "name": "github_token",
  "endpoint": "https://example.com/oauth2/token",
  "client_id": "my-client",
  "client_secret": "shh",
  "min_ttl": "60s"
}
```

- `endpoint`: token URL (required).
- `client_id`, `client_secret`: credentials (required).
- `name`: name the access token is claimed under. Defaults to `token`.
- `min_ttl`: `time.ParseDuration` string. The cached token is
  re-fetched when its remaining lifetime drops to or below this value.
  Omit to refresh only on actual expiry.

Reference as `${oauth:<name>}`. The first `Get` fetches the initial
token; subsequent calls return the cached value until its remaining
lifetime falls under `min_ttl`.

## Servers

Each entry is one upstream MCP. The key is the upstream's name and
becomes the prefix on every tool/prompt/resource it exposes
(`<name>__<original>`). Names must match `[A-Za-z0-9.-]+`.

### stdio

```json
"filesystem": {
  "transport": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
  "env": {
    "LOG_LEVEL": "info"
  }
}
```

### streamable-HTTP

```json
"github": {
  "transport": "streamable",
  "url": "https://api.githubcopilot.com/mcp/",
  "headers": {
    "Authorization": "Bearer ${env:GITHUB_TOKEN}"
  }
}
```

### SSE

```json
"remote": {
  "transport": "sse",
  "url": "https://mcp.example.com/sse",
  "headers": {
    "X-API-Key": "${sops:remote_api_key}"
  }
}
```

## Full example

```json
{
  "proxy": {
    "addr": ":8080",
    "path": "/",
    "transport": "streamable"
  },
  "providers": [
    { "type": "env" },
    {
      "type": "sops",
      "path": "/etc/mcproxy/secrets.enc.json"
    },
    {
      "type": "oauth",
      "name": "api_token",
      "endpoint": "https://auth.example.com/oauth2/token",
      "client_id": "mcproxy",
      "client_secret": "shh",
      "min_ttl": "30s"
    }
  ],
  "servers": {
    "github": {
      "transport": "streamable",
      "url": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer ${env:GITHUB_TOKEN}"
      }
    },
    "internal-api": {
      "transport": "streamable",
      "url": "https://api.example.com/mcp/",
      "headers": {
        "Authorization": "Bearer ${oauth:api_token}"
      }
    },
    "filesystem": {
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
    }
  }
}
```
