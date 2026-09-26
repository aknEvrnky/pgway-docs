# Installation

pgway publishes **GitHub Release** archives and a **GHCR** image when a SemVer tag is pushed (for example `v0.1.0-beta.1`). Until you have a tag you care about, build from source or from the repo `Dockerfile`.

Releases are **manual**: merge to `main`, then tag when ready. Tag pushes run GoReleaser in CI (binaries + `ghcr.io/aknEvrnky/pgway:<version>`). Pre-release tags (`-beta`, `-rc`, …) create GitHub pre-releases automatically. There is no auto-release on every main merge.

## Prerequisites

- [Go](https://go.dev/dl/) **1.27** or newer (`go version`) — not required if you only pull a release binary or image
- Git
- [protoc](https://grpc.io/docs/protoc-installation/) for `make tools` / `make proto` (e.g. `brew install protobuf` or `apt install protobuf-compiler`)
- Optional, only for dashboard development: [Bun](https://bun.sh/) or Node.js 20+ (not required for the gateway itself)
- Optional, for a local image build: [Docker](https://docs.docker.com/get-docker/) (or a compatible engine)

## Clone

```bash
git clone https://github.com/aknEvrnky/pgway.git
cd pgway
```

## Install from a GitHub Release

1. Open the latest (or chosen) release: [aknEvrnky/pgway/releases](https://github.com/aknEvrnky/pgway/releases)
2. Download the archive for your OS/arch (`linux` / `darwin` × `amd64` / `arm64`)
3. Extract and put `pgway`, `pgway-cp`, `pgway-dp`, and/or `pgctl` on your `PATH`

```bash
pgctl version
pgway -version
```

## Install from GHCR

Published images contain all four binaries; default entrypoint is all-in-one `pgway`:

```bash
docker pull ghcr.io/aknEvrnky/pgway:0.1.0-beta.1   # use the tag from the release
docker run --rm --entrypoint /usr/local/bin/pgctl ghcr.io/aknEvrnky/pgway:0.1.0-beta.1 version
```

## Build all binaries (from source)

```bash
make build
```

This writes four binaries under `./build/` and embeds version metadata via ldflags (`VERSION`, git commit, build time):

| Binary | Path |
|--------|------|
| All-in-one | `./build/pgway` |
| Control Plane | `./build/pgway-cp` |
| Data Plane | `./build/pgway-dp` |
| CLI | `./build/pgctl` |

```bash
./build/pgctl version
./build/pgway -version
```

## Docker image (from source)

The repo root [`Dockerfile`](https://github.com/aknEvrnky/pgway/blob/main/Dockerfile) builds one **linux** image that contains all four binaries. The default entrypoint is all-in-one `pgway`. The runtime base is distroless (`gcr.io/distroless/static:nonroot`).

There is **no** official `docker compose` quick-start yet: first-run still needs the bootstrap token from CP stderr and a config file — use [First run](first-run.md) (native binaries) or mount the same files into the container. Distributed CP+DP containers would also need agent registration tokens; that remains a native/[distributed](../guides/distributed.md) flow for now.

### Build

```bash
docker build -t pgway:local .
# optional version metadata:
docker build -t pgway:local \
  --build-arg VERSION=dev \
  --build-arg COMMIT="$(git rev-parse --short HEAD)" \
  --build-arg DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  .
```

### Run all-in-one

Mount a TOML config and a writable Badger data directory. Publish gRPC (and any entrypoint ports you configure):

```bash
docker run --rm \
  -v "$PWD/config.toml:/config/config.toml:ro" \
  -v "$PWD/var/lib:/var/pgway/lib" \
  -p 9090:9090 \
  pgway:local --config /config/config.toml
```

Adjust `badger.path` in the config (or env) so it matches the mounted data dir. Bootstrap admin with `pgctl init` as in [First run](first-run.md) — the one-time token is printed to the container’s **stderr**.

### Other binaries in the same image

```bash
docker run --rm --entrypoint /usr/local/bin/pgctl pgway:local version
docker run --rm --entrypoint /usr/local/bin/pgway-cp pgway:local -version
docker run --rm --entrypoint /usr/local/bin/pgway-dp pgway:local -version
```

## Dev tools and tests

```bash
make tools   # gotestsum, protoc-gen-go, protoc-gen-go-grpc; requires system protoc
make test    # runs the suite via gotestsum (-race)
```

`make test` / `make proto` depend on `make tools`. Put `$(go env GOPATH)/bin` on your `PATH` so `gotestsum` and the `protoc-gen-*` plugins resolve. Install the protobuf compiler separately (`brew install protobuf` or `apt install protobuf-compiler`) — `make tools` verifies `protoc` is available.

## Build a single binary

```bash
go build -o pgway ./cmd/pgway
go build -o pgway-cp ./cmd/pgway-cp
go build -o pgway-dp ./cmd/pgway-dp
go build -o pgctl ./cmd/pgctl
```

## Run without installing

Useful for a quick smoke test from the repo root (still needs a config file — covered in the next Getting Started pages):

```bash
go run ./cmd/pgway
go run ./cmd/pgctl --help
```

Or run a binary you just built:

```bash
./build/pgway
./build/pgctl version
```

## `go install` (optional)

You can install module tip into your `GOBIN` / `GOPATH/bin`:

```bash
go install github.com/aknEvrnky/pgway/cmd/pgway@latest
go install github.com/aknEvrnky/pgway/cmd/pgway-cp@latest
go install github.com/aknEvrnky/pgway/cmd/pgway-dp@latest
go install github.com/aknEvrnky/pgway/cmd/pgctl@latest
```

!!! note
    `@latest` tracks the default branch tip, not a semver release. Prefer a GitHub Release archive or cloning and `make build` when you want a known checkout. `go install` does not inject release ldflags, so `pgctl version` will show `dev` unless you pass your own `-ldflags`.

## What’s next

[Configuration](configuration.md) — config file location, keys, and examples for each binary. Then: bootstrap admin + minimal stack ([First run](first-run.md)).
