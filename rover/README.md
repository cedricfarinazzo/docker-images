# rover

[Apollo Rover CLI](https://www.apollographql.com/docs/rover/) + [Supergraph composition plugin](https://www.apollographql.com/docs/federation/) baked into a single multi-arch image. Drop-in for any federated GraphQL stack that needs `rover supergraph compose` during stack bring-up — no per-deploy CLI download, no plugin warmup.

```bash
docker pull ghcr.io/cedricfarinazzo/rover:latest
```

## Tags

Every release publishes immutable per-combo tags plus rolling aliases.

### Immutable

```
<semver>-rover<X.Y.Z>-sg<X.Y.Z>-debian<codename>
```

Example: `1.0.0-rover0.40.0-sg2.3.0-debianbookworm`.

### Rolling aliases

| Tag | Points to |
|---|---|
| `latest` | newest semver, default combo (rover + supergraph + debian defaults from `versions.json`) |
| `<semver>` | newest combo for that semver |
| `<major>.<minor>` | newest patch in that minor |
| `<major>` | newest minor in that major |
| `latest-debian<codename>` | newest semver on that distro |
| `latest-rover<X.Y.Z>` | newest semver pinning that rover version |
| `latest-sg<X.Y.Z>` | newest semver pinning that supergraph version |

## Architectures

`linux/amd64` + `linux/arm64`. Same manifest. `docker pull` picks the right arch automatically.

## Build args

| Arg | Default | What |
|---|---|---|
| `ROVER_VERSION` | `0.40.0` | Apollo Rover CLI version (pulled from `https://rover.apollo.dev/nix/v<ver>`) |
| `SUPERGRAPH_VERSION` | `2.3.0` | Supergraph plugin version (pulled from `https://rover.apollo.dev/tar/supergraph/<arch>/v<ver>`) |
| `DEBIAN_VERSION` | `bookworm` | Debian codename for base image |

## Downstream usage

### As-is (docker-compose example)

```yaml
rover:
  image: ghcr.io/cedricfarinazzo/rover:latest
  command: ["supergraph", "compose", "--config", "/supergraph.yaml", "--output", "/schema/schema.graphql"]
  volumes:
    - ./apollo-router/supergraph.yaml:/supergraph.yaml:ro
    - schema_composed:/schema
  depends_on:
    api-a: { condition: service_healthy }
    api-b: { condition: service_healthy }
```

The image already sets `APOLLO_ELV2_LICENSE=accept` + `APOLLO_ROVER_SKIP_UPDATE_CHECK=1` — no extra env needed for the standard compose flow.

### Pin a specific combo

```yaml
image: ghcr.io/cedricfarinazzo/rover:1.0.0-rover0.40.0-sg2.3.0-debianbookworm
```

### Smoke test

```bash
docker run --rm ghcr.io/cedricfarinazzo/rover:latest --version
docker run --rm --entrypoint /root/.rover/bin/supergraph-v2.3.0 \
  ghcr.io/cedricfarinazzo/rover:latest --version
```

## Upstream sources

- Apollo Rover CLI: <https://github.com/apollographql/rover>
- Supergraph plugin: shipped with [`@apollo/composition`](https://www.npmjs.com/package/@apollo/composition); binary distributed via `rover.apollo.dev`.

## License

MIT — see repo root [`LICENSE`](../LICENSE).
