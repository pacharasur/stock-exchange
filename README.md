# Stock Exchange Simulator

Educational simulator using fictional instruments and balances only. M0 provides
a minimal matching-engine executable; matching, accounts, persistence, and network
services are not implemented. This is not a real trading system.

## Requirements and modules

Use Go 1.27.3 or later. No third-party dependencies are required.

`go.work` includes only `services/matching-engine`, whose module path is
`github.com/pacharasur/stock-exchange/services/matching-engine`.
The original root `go.mod` is preserved unchanged as the initial module record;
it contains no Go packages and is deliberately excluded from the workspace.
Put service code in its own module, not in the root module.

## Run and verify from the repository root

```sh
go work edit -json
go list -m
go run ./services/matching-engine/cmd/matching-engine
go -C services/matching-engine fmt ./...
go -C services/matching-engine vet ./...
go -C services/matching-engine test ./...
go -C services/matching-engine test -race ./...
go -C services/matching-engine test -cover ./...
go -C services/matching-engine build -o /tmp/stock-exchange-matching-engine ./cmd/matching-engine
```

The executable prints `matching-engine: simulator bootstrap ready` and exits.
Use the explicit service selection above (`go -C` for verification): root `go test ./...` does not select nested
workspace modules when the root directory is not a workspace member.

## Run and verify inside the service

```sh
cd services/matching-engine
go run ./cmd/matching-engine
go fmt ./...
go vet ./...
go test ./...
go test -race ./...
go test -cover ./...
GOWORK=off go test ./...
GOWORK=off go build -o /tmp/stock-exchange-matching-engine ./cmd/matching-engine
```

The `GOWORK=off` commands verify that the service builds independently. The
bootstrap output has an automated example test. There are no domain behaviors
or rejection paths yet; those begin in M1 after review.
