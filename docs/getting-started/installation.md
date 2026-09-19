# Installation

pgway is **not released yet**. There are no versioned GitHub Releases or installers today.

!!! info "Upcoming releases"
    Multi-platform binaries will be published later with [GoReleaser](https://goreleaser.com/) (see issue [#19](https://github.com/aknEvrnky/pgway/issues/19): linux/darwin/windows, amd64/arm64, for `pgway`, `pgway-cp`, `pgway-dp`, and `pgctl`). Until then, run from source.

For now: clone the repository and use **`go build`**, **`make build`**, or **`go run`**.

## Prerequisites

- [Go](https://go.dev/dl/) **1.27** or newer (`go version`)
- Git
- [protoc](https://grpc.io/docs/protoc-installation/) for `make tools` / `make proto` (e.g. `brew install protobuf` or `apt install protobuf-compiler`)
- Optional, only for dashboard development: [Bun](https://bun.sh/) or Node.js 20+ (not required for the gateway itself)

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

[Configuration](configuration.md) — config file location, keys, and examples for each binary. Then: bootstrap admin + minimal stack.
