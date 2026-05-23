ARG ALPINE_VERSION=3.23
FROM alpine:${ALPINE_VERSION}

ARG S6_OVERLAY_VERSION=3.2.1.0

# Install core packages
RUN apk add --no-cache \
      bash \
      ca-certificates \
      curl \
      fcgiwrap \
      git \
      git-daemon \
      gitolite \
      nginx \
      openssh-server \
      openssh-keygen \
      perl \
      shadow \
      tzdata \
      xz \
 && rm -rf /var/cache/apk/*

# Install s6-overlay (arch-aware)
RUN set -eux; \
    ARCH="$(uname -m)"; \
    case "$ARCH" in \
      x86_64)  S6_ARCH="x86_64" ;; \
      aarch64) S6_ARCH="aarch64" ;; \
      *) echo "unsupported arch: $ARCH" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz" -o /tmp/s6-noarch.tar.xz; \
    curl -fsSL "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz" -o /tmp/s6-arch.tar.xz; \
    tar -C / -Jxpf /tmp/s6-noarch.tar.xz; \
    tar -C / -Jxpf /tmp/s6-arch.tar.xz; \
    rm /tmp/s6-noarch.tar.xz /tmp/s6-arch.tar.xz; \
    apk del xz curl

# Create the git user that owns gitolite + serves CGI requests
# Home = /var/lib/gitolite so the volume mount works cleanly.
RUN set -eux; \
    addgroup -S git; \
    adduser -S -G git -h /var/lib/gitolite -s /bin/sh git; \
    mkdir -p /var/lib/gitolite /run/sshd /run/fcgiwrap; \
    chown -R git:git /var/lib/gitolite

# Drop in our config / supervised services / init scripts
COPY rootfs/ /

RUN chmod 0755 \
      /usr/local/bin/gitolite-http-cgi \
      /etc/cont-init.d/01-bootstrap-gitolite \
      /etc/s6-overlay/s6-rc.d/sshd/run \
      /etc/s6-overlay/s6-rc.d/nginx/run \
      /etc/s6-overlay/s6-rc.d/fcgiwrap/run \
 && touch /etc/s6-overlay/s6-rc.d/sshd/type \
          /etc/s6-overlay/s6-rc.d/nginx/type \
          /etc/s6-overlay/s6-rc.d/fcgiwrap/type \
 && touch /etc/s6-overlay/s6-rc.d/user/contents.d/sshd \
          /etc/s6-overlay/s6-rc.d/user/contents.d/nginx \
          /etc/s6-overlay/s6-rc.d/user/contents.d/fcgiwrap \
 && rm -f /etc/nginx/http.d/default.conf

# Persisted state lives here. Mount a volume to survive restarts.
VOLUME ["/var/lib/gitolite", "/etc/ssh/host_keys"]

# 22: SSH (git push/pull, admin)
# 80: HTTP smart-http (git push/pull behind basic-auth). Put TLS in front.
EXPOSE 22 80

ENV S6_KEEP_ENV=1 \
    S6_VERBOSITY=1 \
    GITOLITE_HOME=/var/lib/gitolite \
    GITOLITE_ADMIN_USER=admin

ENTRYPOINT ["/init"]

LABEL org.opencontainers.image.source="https://github.com/cedricfarinazzo/docker-images" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="git-server" \
      org.opencontainers.image.description="Self-hosted git server: gitolite ACLs over SSH + nginx smart-http on alpine ${ALPINE_VERSION}" \
      io.cedricfarinazzo.alpine-version="${ALPINE_VERSION}"
