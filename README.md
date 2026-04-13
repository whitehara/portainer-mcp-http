# portainer-mcp-http

OAuth-protected remote MCP server that wraps [portainer-mcp](https://github.com/portainer/portainer-mcp)
with [mcp-auth-proxy](https://github.com/sigbit/mcp-auth-proxy), using **Cloudflare Access for SaaS**
as the OIDC identity provider. Traffic flows through an existing Traefik reverse proxy and
Cloudflare Tunnel.

## Architecture

```
Claude.ai / Claude Code
    │  mcp-remote (npx)
    ▼
Cloudflare MCP Portal  (https://mcp-portal.your-domain.com/mcp)
    │  OAuth 2.1 bearer token issued by mcp-auth-proxy
    ▼
Cloudflare Tunnel  →  Traefik
    │  Host-based routing (HTTP)
    ▼
┌─────────────────────────────────────────────────┐
│ portainer-mcp-http container (port 80)          │
│                                                  │
│   mcp-auth-proxy  (OAuth server + OIDC client)  │
│        │  stdio                                  │
│        ▼                                         │
│   portainer-mcp                                  │
└──────────────┬──────────────────────────────────┘
               │  REST API
               ▼
         Portainer CE → Docker Swarm

          ▲
          │ OIDC (user login)
          │
Cloudflare Access for SaaS
 (vx-xv.cloudflareaccess.com/.../oidc/<CLIENT_ID>)
```

mcp-auth-proxy acts as both:
- **OAuth 2.1 authorization server** for the Cloudflare MCP Portal (exposes
  `/.well-known/oauth-authorization-server` so the Portal can discover and authenticate)
- **OIDC client** that delegates user identity to Cloudflare Access for SaaS

This means the origin (`portainer-mcp.your-domain.com`) does not need a Cloudflare Access
self-hosted application — authentication is enforced by mcp-auth-proxy itself.

## Prerequisites

- An existing Cloudflare Tunnel connected to your Traefik Docker network
- An existing Traefik container with the Docker provider
- A Cloudflare Zero Trust account
- Docker Compose v2

## Quick start

### 1. Create the Cloudflare Access SaaS (OIDC) app

1. Zero Trust → **Access** → **Applications** → **Add an application** → **SaaS**
2. Authentication protocol: **OIDC**
3. Application name: e.g. `Portainer MCP`
4. Redirect URL: `https://<PORTAINER_MCP_HOSTNAME>/.auth/oidc/callback`
5. Scopes: `openid`, `email`, `profile`
6. Add an Access **Policy** that allows your identity (email / group)
7. Save and copy:
   - **Client ID** → `OIDC_CLIENT_ID`
   - **Client secret** → `OIDC_CLIENT_SECRET`
   - **OIDC discovery URL** → `OIDC_CONFIGURATION_URL`
     (format: `https://<TEAM>.cloudflareaccess.com/cdn-cgi/access/sso/oidc/<CLIENT_ID>/.well-known/openid-configuration`)

### 2. Configure the Tunnel public hostname

Zero Trust → **Networks** → **Tunnels** → your tunnel → **Public Hostnames** → **Add**:

| Field | Value |
|---|---|
| Subdomain / Domain | matches `PORTAINER_MCP_HOSTNAME` |
| Service Type | `HTTP` |
| URL | `traefik:80` (Traefik container name and port) |

### 3. Configure environment and start

```bash
cp .env.example .env
$EDITOR .env   # fill in PORTAINER_*, TRAEFIK_*, OIDC_*, PORTAINER_MCP_HOSTNAME

docker compose pull
docker compose up -d
docker compose logs -f portainer-mcp-http
```

### 4. Register the MCP server in Cloudflare AI controls

1. Zero Trust → **Access controls** → **AI controls** → **MCP servers** → **Add an MCP server**

   | Field | Value |
   |---|---|
   | Name | `Portainer MCP` |
   | HTTP URL | `https://<PORTAINER_MCP_HOSTNAME>/mcp` |

2. Click **Save and connect server** → **Authenticate server**
3. Complete the OAuth popup (authenticates through Cloudflare Access)
4. Status becomes **Ready**

### 5. Add to an MCP Portal (optional but recommended)

Zero Trust → **AI controls** → **Add MCP server portal**

| Field | Value |
|---|---|
| Portal name | `My MCP Portal` |
| Custom domain | `mcp-portal.your-domain.com` |

Add the Portainer MCP server to the portal.

### 6. Configure the client (`.mcp.json`)

```json
{
  "mcpServers": {
    "portainer": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://mcp-portal.your-domain.com/mcp"
      ]
    }
  }
}
```

First launch opens a browser window for Cloudflare Access login. Tokens are cached in
`~/.mcp-auth/`. Reset with `rm -rf ~/.mcp-auth`.

## Environment variables

| Variable | Description |
|---|---|
| `PORTAINER_HOST` | Portainer hostname or IP |
| `PORTAINER_PORT` | Portainer HTTPS port (default: `9443`) |
| `PORTAINER_TOKEN` | Portainer API token (`ptr_...`) |
| `TRAEFIK_NETWORK` | Existing external Docker network shared with Traefik |
| `TRAEFIK_ENTRYPOINT` | Traefik entrypoint name (default: `http`) |
| `PORTAINER_MCP_HOSTNAME` | Public hostname routed by Traefik |
| `OIDC_CLIENT_ID` | OIDC client ID from the Cloudflare SaaS app |
| `OIDC_CLIENT_SECRET` | OIDC client secret |
| `OIDC_CONFIGURATION_URL` | OIDC discovery URL from Cloudflare |
| `OIDC_ALLOWED_USERS_GLOB` | Optional glob allowlist (e.g. `*@example.com`) |
| `IMAGE_TAG` | Image tag to deploy (default: `latest`) |
| `PORTAINER_MCP_VERSION` | *(local build only)* portainer-mcp release version |
| `TARGETARCH` | *(local build only)* `amd64` or `arm64` |

## CI/CD

GitHub Actions (`.github/workflows/build.yml`) builds a multi-arch image
(`linux/amd64` + `linux/arm64`) and pushes to GHCR on push to `main` and on `v*.*.*` tags.

| Event | Tags |
|---|---|
| Push to `main` | `latest`, `main`, `sha-<short>` |
| Push tag `v1.2.3` | `1.2.3`, `1.2`, `sha-<short>` |
| Pull request | build only |

The GHCR package starts as private. Make it public via
`GitHub → package settings → Change visibility → Public` to allow `docker compose pull`
without credentials.

## Upgrade

```bash
docker compose pull portainer-mcp-http
docker compose up -d portainer-mcp-http
docker compose logs -f portainer-mcp-http
```

## Local build

If you need to build locally (for example to bundle a custom `portainer-mcp` version),
comment out `image:` in `docker-compose.yml` and uncomment the `build:` block, then:

```bash
docker compose build --no-cache portainer-mcp-http
docker compose up -d portainer-mcp-http
```
