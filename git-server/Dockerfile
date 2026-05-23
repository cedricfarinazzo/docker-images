ARG ALPINE_VERSION=3.23

# --- builder ---------------------------------------------------------------
# Fetch + verify + extract s6-overlay in a throwaway stage so curl / xz
# never make it into the runtime image.
FROM alpine:${ALPINE_VERSION} AS s6-builder

ARG S6_OVERLAY_VERSION=3.2.1.0
ARG S6_NOARCH_SHA256=42e038a9a00fc0fef70bf0bc42f625a9c14f8ecdfe77d4ad93281edf717e10c5
ARG S6_X86_64_SHA256=8bcbc2cada58426f976b159dcc4e06cbb1454d5f39252b3bb0c778ccf71c9435
ARG S6_AARCH64_SHA256=c8fd6b1f0380d399422fc986a1e6799f6a287e2cfa24813ad0b6a4fb4fa755cc

RUN set -eux; \
    apk upgrade --no-cache; \
    apk add --no-cache curl xz; \
    ARCH="$(uname -m)"; \
    case "$ARCH" in \
      x86_64)  S6_ARCH="x86_64";  ARCH_SHA="$S6_X86_64_SHA256"  ;; \
      aarch64) S6_ARCH="aarch64"; ARCH_SHA="$S6_AARCH64_SHA256" ;; \
      *) echo "unsupported arch: $ARCH" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz"        -o /tmp/s6-noarch.tar.xz; \
    curl -fsSL "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz"   -o /tmp/s6-arch.tar.xz; \
    echo "${S6_NOARCH_SHA256}  /tmp/s6-noarch.tar.xz" | sha256sum -c -; \
    echo "${ARCH_SHA}  /tmp/s6-arch.tar.xz"          | sha256sum -c -; \
    mkdir -p /s6; \
    tar -C /s6 -Jxpf /tmp/s6-noarch.tar.xz; \
    tar -C /s6 -Jxpf /tmp/s6-arch.tar.xz

# --- runtime ---------------------------------------------------------------
FROM alpine:${ALPINE_VERSION}

ARG ALPINE_VERSION

# Pick up upstream alpine patches that landed since alpine:${ALPINE_VERSION}
# was cut, then install runtime deps. No curl/xz/wget/build-* in final.
RUN apk upgrade --no-cache \
 && apk add --no-cache \
      bash \
      ca-certificates \
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
 && rm -rf /var/cache/apk/*

# Drop s6-overlay tree from the builder into / (already extracted, verified).
COPY --from=s6-builder /s6/ /

# Normalise the git user (gitolite apk creates it on pre-install; we only
# need to relocate its home dir + ensure runtime dirs exist).
RUN set -eux; \
    if ! id -u git >/dev/null 2>&1; then \
        addgroup -S git; \
        adduser -S -G git -h /var/lib/gitolite -s /bin/sh git; \
    fi; \
    usermod -d /var/lib/gitolite -s /bin/sh git; \
    mkdir -p /var/lib/gitolite /run/sshd /run/fcgiwrap; \
    chown -R git:git /var/lib/gitolite /run/fcgiwrap; \
    apk del shadow

# Drop in configs / supervised services / init scripts.
COPY rootfs/ /

RUN chmod 0755 \
      /usr/local/bin/gitolite-http-cgi \
      /etc/cont-init.d/01-bootstrap-gitolite \
      /etc/s6-overlay/s6-rc.d/sshd/run \
      /etc/s6-overlay/s6-rc.d/nginx/run \
      /etc/s6-overlay/s6-rc.d/fcgiwrap/run \
 && rm -f /etc/nginx/http.d/default.conf

VOLUME ["/var/lib/gitolite", "/etc/ssh/host_keys"]

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
