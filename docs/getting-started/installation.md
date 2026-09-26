# Installation

pgway is **not released yet**. There are no versioned GitHub Releases or installers today.

!!! info "Upcoming releases"
    Multi-platform binaries and published container images will ship later with [GoReleaser](https://goreleaser.com/) (see issue [#19](https://github.com/aknEvrnky/pgway/issues/19): linux/darwin, amd64/arm64, for `pgway`, `pgway-cp`, `pgway-dp`, and `pgctl`). Until then, run from source or build the Docker image locally.

For now: clone the repository and use **`go build`**, **`make build`**, **`go run`**, or **`docker build`**.

## Prerequisites

- [Go](https://go.dev/dl/) **1.27** or newer (`go version`) — not required if you only build the Docker image
- Git
- [protoc](https://grpc.io/docs/protoc-installation/) for `make tools` / `make proto` (e.g. `brew install protobuf` or `apt install protobuf-compiler`)
- Optional, only for dashboard development: [Bun](https://bun.sh/) or Node.js 20+ (not required for the gateway itself)
- Optional, for the container image: [Docker](https://docs.docker.com/get-docker/) (or a compatible engine)

## Clone

```bash
git clone https://github.com/aknEvrnky/pgway.git
cd pgway
```

## Build all binaries

```bash
make build
```

This writes four binaries under `./build/`:

| Binary | Path |
|--------|------|
| All-in-one | `./build/pgway` |
| Control Plane | `./build/pgway-cp` |
| Data Plane | `./build/pgway-dp` |
| CLI | `./build/pgctl` |

## Docker image (from source)

The repo root [`Dockerfile`](https://github.com/aknEvrnky/pgway/blob/main/Dockerfile) builds one **linux** image that contains all four binaries. The default entrypoint is all-in-one `pgway`. The runtime base is distroless (`gcr.io/distroless/static:nonroot`).

There is **no** official `docker compose` quick-start yet: first-run still needs the bootstrap token from CP stderr and a config file — use [First run](first-run.md) (native binaries) or mount the same files into the container. Distributed CP+DP containers would also need agent registration tokens; that remains a native/[distributed](../guides/distributed.md) flow for now.

Published multi-arch images on a registry land with [#19](https://github.com/aknEvrnky/pgway/issues/19).

### Build

```bash
docker build -t pgway:local .
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
docker run --rm --entrypoint /usr/local/bin/pgctl pgway:local --help
docker run --rm --entrypoint /usr/local/bin/pgway-cp pgway:local --config /config/config.toml
docker run --rm --entrypoint /usr/local/bin/pgway-dp pgway:local --config /config/config.toml
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
./build/pgctl --help
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
    `@latest` tracks the default branch tip, not a semver release. Prefer cloning and `make build` when you want a known local checkout.

## What’s next

[Configuration](configuration.md) — config file location, keys, and examples for each binary. Then: bootstrap admin + minimal stack ([First run](first-run.md)).
