ARG DEBIAN_VERSION=bookworm
FROM debian:${DEBIAN_VERSION}-slim

ARG TARGETARCH
ARG ROVER_VERSION=0.40.0
ARG SUPERGRAPH_VERSION=2.3.0

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl; \
    rm -rf /var/lib/apt/lists/*; \
    \
    # Apollo Rover CLI (pinned)
    curl -sSL "https://rover.apollo.dev/nix/v${ROVER_VERSION}" \
      | APOLLO_ROVER_SKIP_UPDATE_CHECK=1 sh; \
    mv /root/.rover/bin/rover /usr/local/bin/rover; \
    \
    # Supergraph composition plugin (pinned, arch-specific)
    case "${TARGETARCH}" in \
      amd64) ARCH_TRIPLE=x86_64-unknown-linux-gnu ;; \
      arm64) ARCH_TRIPLE=aarch64-unknown-linux-gnu ;; \
      *) echo "Unsupported TARGETARCH: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://rover.apollo.dev/tar/supergraph/${ARCH_TRIPLE}/v${SUPERGRAPH_VERSION}" \
      | tar xz -C /root/.rover/bin/ dist/supergraph; \
    mv /root/.rover/bin/dist/supergraph "/root/.rover/bin/supergraph-v${SUPERGRAPH_VERSION}"; \
    rm -rf /root/.rover/bin/dist; \
    chmod +x "/root/.rover/bin/supergraph-v${SUPERGRAPH_VERSION}"; \
    \
    # Trim build deps
    apt-get purge -y curl; \
    apt-get autoremove -y; \
    rm -rf /var/lib/apt/lists/*

ENV APOLLO_ELV2_LICENSE=accept \
    APOLLO_ROVER_SKIP_UPDATE_CHECK=1

ENTRYPOINT ["rover"]

LABEL org.opencontainers.image.source="https://github.com/cedricfarinazzo/docker-images" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="rover" \
      org.opencontainers.image.description="Apollo Rover CLI + Supergraph composition plugin, multi-arch (amd64 + arm64)."
