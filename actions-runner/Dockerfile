FROM ghcr.io/actions/actions-runner:latest

USER root

ENV DEBIAN_FRONTEND=noninteractive

# ── Core apt packages ────────────────────────────────────────────────────────
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
      binutils \
      build-essential \
      cmake \
      curl \
      dnsutils \
      dpkg-dev \
      fakeroot \
      file \
      findutils \
      g++ \
      gcc \
      gnupg2 \
      iproute2 \
      iputils-ping \
      jq \
      libsqlite3-dev \
      libssl-dev \
      libyaml-dev \
      locales \
      make \
      net-tools \
      netcat-openbsd \
      openssh-client \
      p7zip-full \
      parallel \
      pkg-config \
      python-is-python3 \
      python3 \
      python3-pip \
      python3-venv \
      rsync \
      shellcheck \
      sqlite3 \
      ssh \
      sshpass \
      sudo \
      tar \
      time \
      tree \
      tzdata \
      unzip \
      wget \
      xz-utils \
      zip && \
    rm -rf /var/lib/apt/lists/*

# ── Docker CLI + Compose v2 + Buildx ────────────────────────────────────────
RUN install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
      -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
      https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update -y && \
    apt-get install -y docker-ce-cli docker-compose-plugin docker-buildx-plugin && \
    ln -sf /usr/libexec/docker/cli-plugins/docker-compose /usr/local/bin/docker-compose && \
    rm -rf /var/lib/apt/lists/*

# ── Node.js 22 ───────────────────────────────────────────────────────────────
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    npm install -g yarn && \
    rm -rf /var/lib/apt/lists/*

# ── yq ───────────────────────────────────────────────────────────────────────
ARG TARGETARCH
RUN curl -fsSL "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_${TARGETARCH}" \
      -o /usr/local/bin/yq && \
    chmod +x /usr/local/bin/yq

LABEL org.opencontainers.image.source="https://github.com/cedricfarinazzo/docker-images" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="actions-runner"

USER runner
