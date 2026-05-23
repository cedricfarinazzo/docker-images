# git-server

Self-hosted git server. [gitolite](https://gitolite.com/gitolite/) ACLs over SSH **and** smart-HTTP. No web UI — git transport only. Alpine, multi-arch (`linux/amd64` + `linux/arm64`), supervised by [s6-overlay](https://github.com/just-containers/s6-overlay).

**Registry**: `ghcr.io/cedricfarinazzo/git-server`

## Quick start

```bash
docker run -d \
  --name git \
  -p 2222:22 \
  -p 8080:80 \
  -e GITOLITE_ADMIN_KEY="$(cat ~/.ssh/id_ed25519.pub)" \
  -v git-data:/var/lib/gitolite \
  -v git-host-keys:/etc/ssh/host_keys \
  -v "$PWD/git.htpasswd:/etc/nginx/git.htpasswd:ro" \
  ghcr.io/cedricfarinazzo/git-server:latest
```

On first boot the entrypoint bootstraps gitolite with the pubkey in `GITOLITE_ADMIN_KEY` as the admin user (default `admin`). Subsequent boots ignore the env var — admin/users are managed via the standard gitolite admin-repo workflow.

Clone the admin repo to add users and repos:

```bash
git clone ssh://git@<host>:2222/gitolite-admin
# edit conf/gitolite.conf to declare users + repos
# drop pubkeys in keydir/<username>.pub
git add -A && git commit -m "add alice + new repo" && git push
```

## Authentication

| Transport | Authn | Authz |
|-----------|-------|-------|
| SSH (`git@host:repo.git`) | sshd pubkey (gitolite-managed `authorized_keys`) | gitolite ACL |
| HTTP (`https://host/git/repo.git`) | nginx HTTP basic-auth (`/etc/nginx/git.htpasswd`) | gitolite ACL (`REMOTE_USER` from basic-auth) |

SSH and HTTP enforce the **same** gitolite ACL. The HTTP basic-auth username **must** match the gitolite user (i.e. the `.pub` filename in `keydir/`, without the `.pub` suffix).

### Add an HTTP user

```bash
# on the host (or in any container with htpasswd)
htpasswd -B -c git.htpasswd alice          # first user, creates the file
htpasswd -B    git.htpasswd bob            # add another
# put git.htpasswd at the path you mounted to /etc/nginx/git.htpasswd
docker restart git
```

`alice` and `bob` must also exist as gitolite users (key in `keydir/`).

## Volumes

| Mount | Purpose |
|-------|---------|
| `/var/lib/gitolite` | Repos + gitolite home (admin repo, hooks, conf). Persists everything. |
| `/etc/ssh/host_keys` | sshd host keys (ed25519 / rsa / ecdsa). Generated on first boot, kept across restarts so clients don't get scary "host key changed" warnings. |
| `/etc/nginx/git.htpasswd` | HTTP basic-auth user database. Mount read-only. |

## Ports

| Port | Service |
|------|---------|
| `22/tcp` | sshd (git push/pull, admin) |
| `80/tcp` | nginx smart-http (git push/pull). **Plain HTTP.** Put a reverse proxy in front for TLS. |

## Env

| Var | Default | First-boot required? | Purpose |
|-----|---------|----------------------|---------|
| `GITOLITE_ADMIN_KEY` | — | yes | Admin user's SSH pubkey for gitolite bootstrap. Ignored on subsequent boots. |
| `GITOLITE_ADMIN_USER` | `admin` | no | Admin username (also the `.pub` filename) |
| `GITOLITE_HOME` | `/var/lib/gitolite` | no | gitolite home dir; matches the volume mount |
| `TZ` | `UTC` | no | Container timezone |

## TLS

The image only speaks plain HTTP on port 80. Put a reverse proxy in front (caddy / traefik / nginx / ...). Minimal caddy example:

```caddy
git.example.com {
    reverse_proxy git-server:80
}
```

## Tag scheme

### Immutable per-combo tags

| Pattern | Example |
|---------|---------|
| `<semver>-alpine<ver>`        | `1.0.0-alpine3.23` |
| `<major>.<minor>-alpine<ver>` | `1.0-alpine3.23`   |
| `<major>-alpine<ver>`         | `1-alpine3.23`     |
| `alpine<ver>` (rolling)       | `alpine3.23`       |

### Rolling aliases

| Tag | Points to |
|-----|-----------|
| `latest` | latest semver, default alpine |
| `<semver>`, `<major>.<minor>`, `<major>` | default combo at that semver |
| `latest-alpine<ver>` | latest semver, that alpine version |

### Nightly tags

| Tag | Meaning |
|-----|---------|
| `nightly` | last night's default combo |
| `nightly-alpine<ver>` | rolling per alpine version |
| `nightly-YYYYMMDD-alpine<ver>` | dated immutable nightly snapshot |

## Build args

| Arg                  | Default   | Description                                  |
|----------------------|-----------|----------------------------------------------|
| `ALPINE_VERSION`     | `3.23`    | Alpine base tag (`alpine:${ALPINE_VERSION}`) |
| `S6_OVERLAY_VERSION` | `3.2.1.0` | s6-overlay release pulled from upstream      |

## Build locally

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f git-server/Dockerfile \
  --build-arg ALPINE_VERSION=3.23 \
  -t git-server:test \
  git-server/
```

## Security notes

- sshd is **pubkey-only**. Password auth disabled at the config level. Only the `git` user is allowed to log in (`AllowUsers git`).
- HTTP is plain text. Run behind a TLS-terminating reverse proxy.
- `git.htpasswd` should use bcrypt (`htpasswd -B`) — nginx's `auth_basic` accepts it.
- Volumes hold private state (repos, host keys, htpasswd) — restrict host filesystem perms accordingly.
- The container does not currently run sshd as a non-root user. sshd needs port 22 + setuid for the auth handshake; standard for self-hosted git server images.

## Upstream sources

- gitolite: <https://gitolite.com/gitolite/>
- s6-overlay: <https://github.com/just-containers/s6-overlay>
- Alpine base: <https://hub.docker.com/_/alpine>

## License

MIT. gitolite is GPL-2.0-or-later; nginx is BSD-2-Clause; openssh is BSD; s6-overlay is ISC. See upstream.
