# -----------------------------------------------
# Stage 1: Download and verify portainer-mcp binary
# -----------------------------------------------
FROM alpine:3.19 AS downloader

ARG PORTAINER_MCP_VERSION=v0.7.0
ARG TARGETARCH=amd64

WORKDIR /tmp

RUN wget -O portainer-mcp.tar.gz \
      "https://github.com/portainer/portainer-mcp/releases/download/${PORTAINER_MCP_VERSION}/portainer-mcp-${PORTAINER_MCP_VERSION}-linux-${TARGETARCH}.tar.gz" \
 && wget -O portainer-mcp.tar.gz.md5 \
      "https://github.com/portainer/portainer-mcp/releases/download/${PORTAINER_MCP_VERSION}/portainer-mcp-${PORTAINER_MCP_VERSION}-linux-${TARGETARCH}.tar.gz.md5"

RUN EXPECTED=$(cat portainer-mcp.tar.gz.md5) \
 && ACTUAL=$(md5sum portainer-mcp.tar.gz | awk '{print $1}') \
 && echo "Expected: ${EXPECTED}" \
 && echo "Actual:   ${ACTUAL}" \
 && if [ "${EXPECTED}" != "${ACTUAL}" ]; then \
      echo "ERROR: md5sum mismatch. Aborting." && exit 1; \
    fi \
 && echo "md5sum OK"

RUN tar -xzf portainer-mcp.tar.gz \
 && chmod +x portainer-mcp

# -----------------------------------------------
# Stage 2: Runtime image based on mcp-auth-proxy
# -----------------------------------------------
FROM ghcr.io/sigbit/mcp-auth-proxy:latest

COPY --from=downloader /tmp/portainer-mcp /usr/local/bin/portainer-mcp

RUN mkdir -p /etc/portainer-mcp /data

# All CLI args (OIDC config, --, portainer-mcp args) are passed via command: in docker-compose.yml
# mcp-auth-proxy's default ENTRYPOINT is used as-is.
