# portainer-mcp-http

Expose [portainer-mcp](https://github.com/portainer/portainer-mcp) as an MCP server via
[mcp-proxy](https://github.com/sparfenyuk/mcp-proxy), routed through an existing Traefik reverse
proxy and protected by Cloudflare Zero Trust Access.

## Architecture

```
Claude.ai / Claude Code
    │  mcp-remote (npx)
    ▼
Cloudflare MCP Portal  (https://mcp-portal.your-domain.com/mcp)
    │  Cloudflare Zero Trust Access (IdP authentication)
    ▼
Cloudflare Tunnel (existing cloudflared container)
    │
    ▼
Traefik (existing, HTTP)
    │  Host-based routing
    ▼
portainer-mcp-http container  (mcp-proxy + portainer-mcp, port 8080)
    │  REST API
    ▼
Portainer CE → Docker Swarm
```

## Prerequisites

- An existing Cloudflare Tunnel with a `cloudflared` container running and connected to a Traefik
  Docker network
- An existing Traefik container configured with the Docker provider
- A Cloudflare account with Zero Trust Access enabled
- Docker Compose v2

## CI/CD — GitHub Actions

The included workflow (`.github/workflows/build.yml`) builds a multi-arch image
(`linux/amd64` + `linux/arm64`) and pushes it to GHCR on every push to `main` and on version
tags (`v*.*.*`). Pull requests trigger a build-only run (no push).

### Image tags

| Event | Tags produced |
|---|---|
| Push to `main` | `latest`, `main`, `sha-<short>` |
| Push tag `v1.2.3` | `1.2.3`, `1.2`, `sha-<short>` |
| Pull request | build only, no push |

### Bundling a specific portainer-mcp version

Trigger **workflow_dispatch** from the Actions tab and set
`portainer_mcp_version` (e.g. `v0.8.0`). Leave blank to use the Dockerfile default.

### Making the image public

By default GHCR packages are private. To allow `docker compose pull` without credentials:

1. GitHub → `whitehara/portainer-mcp-http` → **Packages** → select the package
2. **Package settings** → **Change visibility** → Public

## Directory Structure

```
portainer-mcp-http/
├── .github/
│   └── workflows/
│       └── build.yml         # GHCR build & push
├── docker-compose.yml
├── .env                      # secrets — never commit
├── .env.example
├── .gitignore
└── portainer-mcp-http/
    └── Dockerfile
```

## Setup

### 1. Prepare environment variables

```bash
cp .env.example .env
$EDITOR .env   # fill in all values
```

| Variable | Description |
|---|---|
| `PORTAINER_HOST` | Portainer hostname or IP |
| `PORTAINER_PORT` | Portainer HTTPS port (default: `9443`) |
| `PORTAINER_TOKEN` | Portainer API token (`ptr_...`) |
| `TRAEFIK_NETWORK` | Name of the existing external Docker network shared with Traefik |
| `TRAEFIK_ENTRYPOINT` | Traefik entrypoint name for HTTP traffic (default: `http`) |
| `PORTAINER_MCP_HOSTNAME` | Public hostname Traefik will route to this service |
| `IMAGE_TAG` | Image tag to deploy (default: `latest`) |
| `PORTAINER_MCP_VERSION` | *(local build only)* portainer-mcp release version |
| `TARGETARCH` | *(local build only)* binary architecture: `amd64` or `arm64` |

> **Tip:** Use a non-guessable subdomain for `PORTAINER_MCP_HOSTNAME`
> (e.g. `mcp-origin-a7f2k3.your-domain.com`) to minimise exposure during the Cloudflare
> Access setup window.

### 2. Configure Cloudflare Zero Trust Access

> **⚠️ CRITICAL ORDER OF OPERATIONS**
>
> Create the Cloudflare Zero Trust Access application **before** adding the Public Hostname
> to your Tunnel. A Public Hostname creates a live DNS record immediately — if Access is not
> already in place, the endpoint is publicly reachable without authentication until you add it.
>
> Order:
> 1. Create the Access application (step below) — DNS does not exist yet, that is fine
> 2. Add the Public Hostname to the Tunnel
> 3. Start the portainer-mcp-http container

#### 2a. Create the Access application

1. [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → **Access** → **Applications**
   → **Add an application** → **Self-hosted**
2. Set **Application domain** to your `PORTAINER_MCP_HOSTNAME` value
3. Add a **Policy** that allows only your identity (e.g. your email address)
4. Configure your preferred **Login method** (Google, GitHub, etc.)
5. Save

#### 2b. Add the Public Hostname to your Tunnel

1. Zero Trust → **Networks** → **Tunnels** → select your existing tunnel
2. **Public Hostnames** → **Add a public hostname**

   | Field | Value |
   |---|---|
   | Subdomain / Domain | matches `PORTAINER_MCP_HOSTNAME` |
   | Service Type | `HTTP` |
   | URL | `traefik:80` (or the container name / port of your Traefik instance) |

3. Save — Cloudflare will create the DNS CNAME automatically

#### 2c. Register as an MCP Server

1. Zero Trust → **Access controls** → **AI controls** → **MCP servers** → **Add an MCP server**

   | Field | Value |
   |---|---|
   | Name | `Portainer MCP` |
   | HTTP URL | `https://<PORTAINER_MCP_HOSTNAME>/mcp` |

2. Set **Access policies** to match the application created in step 2a
3. **Save and connect server** — wait for status to show **Ready**

#### 2d. Create an MCP Server Portal (optional but recommended)

1. **Add MCP server portal**

   | Field | Value |
   |---|---|
   | Portal name | `My MCP Portal` |
   | Custom domain | `mcp-portal.your-domain.com` |

2. Add the Portainer MCP server to the portal

### 3. Pull and start

```bash
# Pull the pre-built image from GHCR
docker compose pull

# Start
docker compose up -d

# Verify logs
docker compose logs -f portainer-mcp-http
```

> **Local build fallback:** If you want to build locally instead of pulling from GHCR,
> comment out the `image:` line in `docker-compose.yml` and uncomment the `build:` block,
> then run `docker compose build` before `docker compose up -d`.

### 4. Configure the client

#### Claude Code (`.mcp.json`)

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

First launch will open a browser for IdP authentication. The token is cached in `~/.mcp-auth/`
and reused until the session expires.

To reset cached credentials:

```bash
rm -rf ~/.mcp-auth
```

## Upgrade

### Via GitHub Actions (recommended)

1. Trigger **workflow_dispatch** from the Actions tab with the new `portainer_mcp_version`
   (e.g. `v0.8.0`) — or update the `ARG PORTAINER_MCP_VERSION` default in the Dockerfile and
   push to `main`
2. Wait for the build to complete and the new image to appear in GHCR
3. On the host, pull and restart:

```bash
docker compose pull portainer-mcp-http
docker compose up -d portainer-mcp-http
docker compose logs -f portainer-mcp-http
```

### Local build

```bash
# 1. Switch docker-compose.yml to build: mode (comment out image:, uncomment build:)
# 2. Update PORTAINER_MCP_VERSION in .env

# 3. Rebuild (md5 verification runs automatically)
docker compose build --no-cache portainer-mcp-http

# 4. Restart
docker compose up -d portainer-mcp-http
docker compose logs -f portainer-mcp-http
```

## Troubleshooting

### md5 checksum mismatch

```
ERROR: md5sum mismatch. Aborting.
```

Re-run with `--no-cache`. If the error persists, verify the release files on the
[portainer-mcp releases page](https://github.com/portainer/portainer-mcp/releases).

### Version mismatch warning

```
error: portainer version mismatch
```

`-disable-version-check` is already included in `docker-compose.yml`. If you see this after
a Portainer upgrade, consider updating `PORTAINER_MCP_VERSION` to match.

### Traefik not routing to the container

- Confirm the container has joined the correct external network: `docker network inspect <TRAEFIK_NETWORK>`
- Confirm Traefik has the Docker provider enabled and is watching the same network
- Check Traefik logs: `docker logs <traefik-container>`

### Cloudflare Tunnel not forwarding

```bash
docker logs cloudflared-mcp
```

Verify the Public Hostname service URL points to Traefik (not directly to `portainer-mcp-http`).

### MCP Server Portal status shows Error

Zero Trust → **AI controls** → **MCP servers** → select server → **Authenticate server** to
re-trigger authentication.
