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
# Stage 2: Runtime image using mcp-proxy
# -----------------------------------------------
FROM ghcr.io/sparfenyuk/mcp-proxy:latest

# portainer-mcp is a glibc-linked binary; Alpine uses musl, so install gcompat
RUN apk add --no-cache gcompat

COPY --from=downloader /tmp/portainer-mcp /usr/local/bin/portainer-mcp

RUN mkdir -p /etc/portainer-mcp

# Actual -server, -token, and other args are overridden via command: in docker-compose.yml
ENTRYPOINT ["mcp-proxy"]
CMD ["--host=0.0.0.0", "--port=8080", \
     "--", \
     "/usr/local/bin/portainer-mcp", \
     "-server", "placeholder", \
     "-token",  "placeholder", \
     "-tools",  "/etc/portainer-mcp/tools.yaml"]
