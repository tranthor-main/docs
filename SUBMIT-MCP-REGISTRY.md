# Submit the docs Model Context Protocol (MCP) server to the official registry

This is a **draft**. Do not run `mcp-publisher publish` until the owner has merged the docs fixes, confirmed they are live on `https://docs.tranthor.com`, and completed domain verification.

The official registry is https://github.com/modelcontextprotocol/registry. Publishing docs are at https://modelcontextprotocol.io/registry/quickstart and https://modelcontextprotocol.io/registry/remote-servers.

## How the docs MCP server is served

This repository does **not** implement an MCP server. There is no custom `/mcp` route, Worker, or package here.

Mintlify hosts a search MCP server for the docs site:

- Public Hypertext Transfer Protocol (HTTP) endpoint: `https://docs.tranthor.com/mcp`
- Transport: streamable HTTP (Mintlify's discovery document reports `"transport": "http"`)
- Discovery: `GET https://docs.tranthor.com/.well-known/mcp`
- This repo enables the docs-site MCP control by including `"mcp"` in `docs.json` under `contextual.options`
- Mintlify reserves `/mcp` and `/authed/mcp`. Do not add a docs page at those paths

The live site is Mintlify on Vercel (`x-mintlify-client-version`, `server: Vercel`). Connecting this GitHub repo in the Mintlify dashboard is what would make **this** repository the content source. Today the published site still serves the existing Tranthor docs tree, which is not fully present in this starter checkout.

A separate product MCP server exists at `https://api.tranthor.com/mcp` (workspace tools). That is not the docs server. Do not register that URL as `com.tranthor/docs`.

## Draft metadata

The draft `server.json` at the repo root follows the current schema (`https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json`):

- Name: `com.tranthor/docs` (reverse Domain Name System (DNS) of `tranthor.com`, plus `/docs`)
- Remote: streamable HTTP at `https://docs.tranthor.com/mcp`
- No `packages` entry (remote-only)

The registry description field is limited to 100 characters, so "API" is not expanded there.

**TODO (owner):** Confirm the name `com.tranthor/docs`. If you prefer `io.github.tranthor-main/docs`, you can skip domain verification and log in with GitHub instead. Domain namespacing is the better match for a remote URL on `docs.tranthor.com`.

## What must be true before anyone publishes

1. The WooCommerce docs pull request is merged to the default branch.
2. Those docs are actually live on `https://docs.tranthor.com` (Mintlify is deploying from this repo, or the same content is published another way).
3. `https://docs.tranthor.com/mcp` still responds to a streamable HTTP `initialize` request (see the check below).
4. You control `tranthor.com` and have finished DNS or HTTP verification for the `com.tranthor` namespace.
5. No other registry server already uses the remote URL `https://docs.tranthor.com/mcp`.
6. You have reviewed `server.json` (name, description, version).

Do not publish from a laptop that only has this draft branch. Do not publish to npm, PyPI, Docker Hub, or any other package registry. This server is remote-only.

## Check that the endpoint responds

Checked on 2 November 2026 from this environment:

| Request | Result |
| --- | --- |
| `GET https://docs.tranthor.com/.well-known/mcp` | HTTP 200. JSON names the public server at `https://docs.tranthor.com/mcp` with authentication `none`. |
| `GET https://docs.tranthor.com/mcp` | HTTP 405. JSON-RPC error `Method not allowed.` (expected for streamable HTTP; clients must POST) |
| `POST https://docs.tranthor.com/mcp` with `initialize` | HTTP 200, `content-type: text/event-stream`. `serverInfo.name` is `Tranthor`, `version` is `1.0.0`. Tools and resources are advertised. |

Re-check after the docs go live:

```bash
curl -sS -D - -X POST 'https://docs.tranthor.com/mcp' \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'MCP-Protocol-Version: 2025-03-26' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"tranthor-docs-check","version":"0.0.1"}}}'
```

You should see HTTP 200 and an `event: message` payload with `serverInfo`.

## Domain verification for `com.tranthor/docs`

GitHub login can only publish names like `io.github.tranthor-main/docs`. For `com.tranthor/docs` you must prove control of `tranthor.com` with DNS or HTTP. The registry checks the **name namespace**, not that the remote URL is on the same domain. Prefer a name under `tranthor.com` anyway, so users see the same publisher as `docs.tranthor.com`.

Store the private key outside git. Do not commit `key.pem`.

### Option A: DNS (TXT on `tranthor.com`)

```bash
MY_DOMAIN="tranthor.com"

openssl genpkey -algorithm Ed25519 -out key.pem

PUBLIC_KEY="$(openssl pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
echo "${MY_DOMAIN}. IN TXT \"v=MCPv1; k=ed25519; p=${PUBLIC_KEY}\""
```

Add that TXT record at your DNS provider. Wait until it resolves, then:

```bash
MY_DOMAIN="tranthor.com"
PRIVATE_KEY="$(openssl pkey -in key.pem -noout -text | grep -A3 "priv:" | tail -n +2 | tr -d ' :\n')"
mcp-publisher login dns --domain "${MY_DOMAIN}" --private-key "${PRIVATE_KEY}"
```

### Option B: HTTP (file on `tranthor.com`)

```bash
openssl genpkey -algorithm Ed25519 -out key.pem

PUBLIC_KEY="$(openssl pkey -in key.pem -pubout -outform DER | tail -c 32 | base64)"
echo "v=MCPv1; k=ed25519; p=${PUBLIC_KEY}" > mcp-registry-auth
```

Host that file at `https://tranthor.com/.well-known/mcp-registry-auth` (the registrable domain, not `docs.tranthor.com`, unless you also change the server name to `com.tranthor.docs/...`). Then:

```bash
MY_DOMAIN="tranthor.com"
PRIVATE_KEY="$(openssl pkey -in key.pem -noout -text | grep -A3 "priv:" | tail -n +2 | tr -d ' :\n')"
mcp-publisher login http --domain "${MY_DOMAIN}" --private-key "${PRIVATE_KEY}"
```

Cloud signing (Google Key Management Service or Azure Key Vault) is documented at https://modelcontextprotocol.io/registry/authentication if you do not want a local private key.

## Exact `mcp-publisher` steps (after the gates above)

1. Install the publisher (pick one):

```bash
brew install mcp-publisher
```

```bash
curl -L "https://github.com/modelcontextprotocol/registry/releases/latest/download/mcp-publisher_$(uname -s | tr '[:upper:]' '[:lower:]')_$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/').tar.gz" | tar xz mcp-publisher
sudo mv mcp-publisher /usr/local/bin/
```

2. Confirm the binary:

```bash
mcp-publisher --help
```

3. From this repository root, validate the draft. Do not publish yet:

```bash
mcp-publisher validate ./server.json
```

You want `server.json is valid`.

4. Authenticate with DNS or HTTP as in the previous section (`mcp-publisher login dns ...` or `mcp-publisher login http ...`).

5. Only after steps 1 through 6 in **What must be true**, publish:

```bash
mcp-publisher publish ./server.json
```

6. Confirm:

```bash
curl "https://registry.modelcontextprotocol.io/v0.1/servers?search=com.tranthor/docs"
```

If publish fails with a namespace error, the login method does not match `com.tranthor/docs`. If it fails because the remote URL is taken, another server already registered `https://docs.tranthor.com/mcp`.

## What this draft does not do

- It does not submit to the registry.
- It does not merge, deploy, or publish the docs site.
- It does not put a custom MCP implementation in this repo. Mintlify keeps hosting `/mcp`.
