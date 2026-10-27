# Stock Exchange Simulator — Codex Project Guide

> This file is intended to live at the **repository root** as `AGENTS.md`.
> Codex should read it before each implementation task, and use its checklists as the source of truth for progress.
> Status at handoff: only the directory creation commands below were confirmed by the user. **Do not assume any implementation or test has passed.**

## 1. Mission and boundaries

Build an educational **stock exchange simulator** with a correct, deterministic, in-memory central limit order book (CLOB), then incrementally introduce durability, account/asset reservation, settlement, market data, and independently deployable services.

Primary learning goals: **high-performance systems** (data structures, latency, throughput) and **distributed systems** (ordering, idempotency, consistency, replay, failover).

- Simulator only: fictional instruments and balances; **no real orders, money, custodial assets, broker connectivity, or production trading claims**.
- Start with `LIMIT` orders and `GTC`; add advanced order types only after matching correctness is proved.
- Prefer **Go** for backend services, **React + TypeScript** for a future UI, **PostgreSQL** for transactional state, **Kafka** for integration events, **Redis** for disposable read-side caches, **Docker Compose** for local development, and later Prometheus/Grafana/OpenTelemetry.
- Make the core correct before optimizing it. A clean benchmark does not prove transaction durability.
- Treat “production-style engineering” as a quality goal, not evidence that the simulator is legally or operationally ready for a real exchange.

## 2. Current verified context

- Developer's machine: macOS ARM64, Go `1.27.3`.
- GitHub username: `pacharasur`.
- An initial root `go.mod` was created with module `github.com/pacharasur/stock-exchange` and `go 1.27.3`.
- User reported running the following commands successfully:

```bash
mkdir -p services/matching-engine/cmd/matching-engine
mkdir -p services/matching-engine/internal/engine
mkdir -p services/matching-engine/internal/orderbook
mkdir -p services/matching-engine/internal/order
mkdir -p services/matching-engine/internal/trade
```

- **Unknown**: whether root-level `cmd/`/`internal/` files were created, whether any code exists, Git status, whether service-level `go.mod` or `go.work` exists. Inspect before changing anything.
- The user's Mac workspace is not automatically accessible from this conversation. Codex in VS Code must inspect that actual workspace.

## 3. Architectural decisions

### Repository layout (target, not an instruction to create empty directories immediately)

```text
stock-exchange/
├── AGENTS.md
├── README.md
├── go.work                       # local Go multi-module workspace
├── services/
│   ├── matching-engine/
│   │   ├── go.mod                 # independently buildable module
│   │   ├── cmd/matching-engine/main.go
│   │   └── internal/
│   │       ├── order/
│   │       ├── orderbook/
│   │       ├── engine/
│   │       └── trade/
│   ├── order-gateway/            # future
│   ├── ledger/                   # future
│   ├── market-data/              # future
│   └── portfolio/                # future, can begin as a read model
├── contracts/                    # versioned schemas, future
├── infrastructure/               # Docker Compose, telemetry, future
└── docs/                         # ADRs, API and architecture, future
```

### Service boundaries

1. **Matching engine** owns the deterministic order-book state machine and matching rules. Single writer per symbol partition; no per-match remote calls or database queries in the hot path. In-memory state is **not** sufficient durability.
2. **Order gateway** (future) handles authentication, validation, idempotency, instrument rules and admission; successful HTTP admission is distinct from exchange acceptance and fill.
3. **Risk and reservation** (future) must atomically prevent double-spending of cash or shares. It can start inside the ledger/order gateway transactional boundary; do not distribute this critical invariant prematurely.
4. **Ledger/settlement** (future) owns balanced, append-only asset transfers with deduplicated execution events and reconciliation. Separate available from reserved amounts.
5. **Market data** (future) publishes snapshots and sequenced deltas, with gap detection and resynchronization.
6. **Kafka** is for fan-out/integration, **not** a substitute for a defined durable commit protocol. Decide what acknowledgement means and what can be replayed before claiming crash safety.

### Go design rules

- `cmd/<binary>/main.go` is composition/bootstrap only: config, dependencies, lifecycle, shutdown. No matching business logic there.
- `internal/` contains implementation packages; organize around responsibilities, not mandatory `controller/service/repository` layers.
- Avoid unnecessary interfaces; introduce them at a real dependency/test boundary.
- Use integer `PriceTicks` and integer share quantities, never floating-point money. Enforce symbol-specific tick-size and quantity rules when instruments are introduced.
- Use explicit error handling and validation; no panics for invalid user orders.
- Keep matching engine deterministic given the same ordered commands and initial state; do not use wall-clock timing to rank orders. Sequence numbers come from the sequencer/command input.
- Do not launch goroutines that mutate the same order book concurrently. Partition by symbol only after correctness is established.
- Tests must check invariants, not only successful examples.

## 4. Working protocol for Codex

1. Inspect `pwd`, `rg --files`, `git status --short`, existing `go.mod`/`go.work`, and relevant source before proposing edits.
2. Work on **one unchecked milestone or a small task within it**. State intended files and behavior before coding. Don't implement the entire roadmap in one request.
3. Preserve pre-existing user code and unrelated changes. No deleting/moving/replacing the existing root module until confirming usage and obtaining user agreement for the migration.
4. Add or update automated tests for each behavior. Keep domain tests free of database/Kafka dependencies.
5. Run the applicable verification commands. Record their real results; never mark a checkbox complete based only on code appearance.
6. Update the progress checklist in this file when a milestone's acceptance criteria **actually pass**. Include commit hash if available, tests run, and remaining risks in the progress log.
7. When blocked or a choice changes core semantics (execution-price rule, cancellation races, commit durability), stop and present alternatives before implementing.
8. End each implementation turn with: changed files, tests/results, remaining gaps, next smallest task.

## 5. Initial setup: migrate to multi-module safely

**Do not run destructive operations blindly.** Initial `go.mod` may still exist in the repository root, which means nesting a service-level module underneath it affects package discovery.

- [x] **M0.1 Inventory:** inspect root `go.mod`, any Go files, `go.work`, `.gitignore`, and Git status. Identify anything that would be affected by migration.
- [x] **M0.2 Agree migration:** if root Go code exists, agree where it moves; preserve history/content. Make an independent matching-engine module at `services/matching-engine/go.mod` with path `github.com/pacharasur/stock-exchange/services/matching-engine`.
- [x] **M0.3 Workspace:** after root module is retired safely (or deliberately retained with documented purpose), add `go.work` using the matching-engine module. Do not accidentally use root `go.mod` as a catch-all for services.
- [x] **M0.4 Bootstrap:** create `cmd/matching-engine/main.go` that launches a minimal executable; create README with exact run/test commands; create a focused `.gitignore` (build output, IDE files, secrets, local data, coverage).
- [x] **M0.5 Verify:** inside `services/matching-engine`, `go run ./cmd/matching-engine`, `go test ./...`, `go vet ./...`; from root, verify `go work` and `go work`-aware commands behave as documented.

**M0 acceptance:** module paths are correct; one matching-engine binary runs; no duplicate/conflicting source trees; no user data lost; verification passes.

## 6. Milestone roadmap and acceptance criteria

### M1 — Domain model and matching rules (first coding milestone)

- [ ] Define `OrderID`, `UserID`/`AccountID`, `Symbol`, `Side` (`BUY`/`SELL`), `OrderType` (`LIMIT`), `TimeInForce` (`GTC`), `PriceTicks`, `Quantity`, `RemainingQuantity`, `Sequence`, and explicit states.
- [ ] Validate nonzero IDs, allowed enums, known symbol (once instrument registry exists), positive integer quantities/prices, `Remaining <= Quantity`, and duplicate IDs.
- [ ] Specify execution price as the **resting order's limit price**; bids execute when incoming buy >= best ask, asks when incoming sell <= best bid.
- [ ] Define exact deterministic ordering: highest bid first; lowest ask first; FIFO by engine-assigned sequence within one price level.
- [ ] Define partial fill, cancel, post-resting remainder, and no-cross outcomes. A filled/cancelled order cannot be filled again.
- [ ] Write table-driven tests for validation and the trading rules; use no DB, Kafka or HTTP.

**M1 acceptance:** documented behavioral contract and passing tests for valid/invalid orders and expected trade outcomes.

### M2 — In-memory order book and functional engine

- [ ] Implement bid/ask price levels with FIFO queues and an ID index for cancellation. Prefer the simplest correct structure first; document actual time complexity.
- [ ] Implement order submission, matching, partial fills, cancellation, order status changes and generated trades/executions.
- [ ] Support one incoming order matching multiple resting orders and multiple price levels.
- [ ] Reject duplicate commands/order IDs deterministically; ensure no negative remaining quantity, duplicate fill, or crossed book after processing.
- [ ] Expose a read-only depth snapshot (aggregate quantity per level); snapshots must not allow callers to mutate the book.
- [ ] Add adversarial tests: equal-price FIFO, multiple levels, no match, exact match, partial fill, cancel before/after fill, duplicate ID, invalid quantity, empty book.
- [ ] Add a deterministic simulation CLI and benchmark harness (separate from correctness tests).

**M2 acceptance:** `go test ./...`, `go test -race ./...`, and replaying a fixed command stream produce matching outcomes and final book state. Benchmarks report hardware and workload without unverified performance claims.

### M3 — Durability and recovery design

- [ ] Write ADR specifying authoritative command/event journal, sequence assignment, commit/ack boundary, checksums, and replay semantics.
- [ ] Implement durable append-only journal or an explicitly scoped transactional prototype; don't use plain in-memory state as the only truth.
- [ ] Implement versioned snapshots and replay from journal tail; verify checksums and sequence continuity.
- [ ] Test crashes at each significant boundary, duplicate delivery, torn/corrupt journal records, restart and exact final-state recovery.

**M3 acceptance:** restart reproduces the same book/trades and no acknowledged committed result is lost in the fault-injection scenarios defined by the ADR.

### M4 — Order gateway, accounts, risk and ledger

- [ ] Add gateway HTTP endpoints for submit/cancel/status with explicit idempotency keys and order state transitions.
- [ ] Create PostgreSQL migrations for accounts, balances/reservations, orders, trades, ledger transactions/entries and outbox; include constraints/indexes.
- [ ] Support test-money and test-share initial allocation. Reserve available cash/shares atomically before admitting an order; reject insufficient balance.
- [ ] Apply fills with balanced, append-only double-entry asset transfers and unique execution IDs; release excess or cancelled reservations safely.
- [ ] Address cancellation-versus-fill races via sequenced commands and finalized execution outcomes.
- [ ] Add transactional integration tests for concurrent orders, double spending, retry, duplicate event, cancellation and reconciliation.

**M4 acceptance:** no negative spendable balances, no duplicate trade settlement, balances reconcile per asset; API returns meaningful acceptance vs execution status.

### M5 — Events, market data and UI

- [ ] Define versioned event schemas with event ID, symbol, symbol sequence, trade/order IDs, and event time.
- [ ] Publish only committed events via a documented journal/outbox bridge. Consumers are idempotent.
- [ ] Implement WebSocket book snapshot + delta protocol with sequence-gap detection/resync; trades/ticker channels.
- [ ] Build React + TypeScript demo: submit/cancel, live depth, order history, trades and simulated portfolio.
- [ ] Test reconnect, out-of-order/duplicate delivery, stale data and slow consumers (backpressure).

**M5 acceptance:** end-to-end simulated trade is visible in UI; restarting consumers does not double-apply balances/trades; gaps trigger safe resync.

### M6 — Performance, scaling and operations

- [ ] Record baseline throughput and p50/p95/p99 latency separately for in-memory engine, journaled engine and end-to-end API.
- [ ] Profile allocation, GC, CPU and memory; optimize only measured bottlenecks.
- [ ] Partition by symbol with exactly one active writer per partition and a documented ownership/fencing/failover protocol.
- [ ] Add structured logging, metrics, tracing, health probes, graceful shutdown, Docker Compose and CI.
- [ ] Run controlled load and fault-injection tests; document test environment and reproducible results.

**M6 acceptance:** documented benchmark methodology, operational runbook, failure/recovery test matrix and clean CI with meaningful latency/throughput measurements.

### M7 — Advanced market behavior (optional, after M6)

- [ ] Trading sessions, halt/resume and market rules.
- [ ] IOC/FOK and protected market orders with explicit semantics and tests.
- [ ] Self-trade prevention, surveillance/audit views and FIX gateway exploration.
- [ ] Evaluate Rust/C++ engine prototype against Go under the same deterministic test corpus and benchmark workload.

## 7. Verification checklist for every milestone

Use the commands appropriate to the active Go module (typically `cd services/matching-engine`):

```bash
go fmt ./...
go vet ./...
go test ./...
go test -race ./...
go test -cover ./...
```

- [x] All applicable commands complete successfully (report any skipped/not applicable checks).
- [x] All specified behavior has automated tests, including rejection/error paths. M0: bootstrap output example test; no order inputs or rejection paths yet.
- [x] No shared mutable book without serialization, no float prices, no DB/network calls inside matching loop.
- [x] No secrets or user-specific absolute paths added; documented commands verified locally. Fresh-clone check not applicable: no Git repository exists.
- [x] README/ADRs/API contracts updated where behavior or architecture changed.
- [x] Logs don't expose authentication tokens or financial secrets.
- [x] Scope and known limitations explicitly documented; never call the project real-exchange-production-ready.

## 8. Progress log (Codex must update with evidence)

| Milestone | Status | Evidence / verified commands | Notes |
| --- | --- | --- | --- |
| M0 Setup | COMPLETE (2026-10-27) | Go 1.27.3 darwin/arm64: service `go fmt ./...`, `go vet ./...`, `go test ./...`, `go test -race ./...`, `go test -cover ./...` passed (bootstrap coverage 100%); `go run ./cmd/matching-engine` passed; `GOWORK=off go test ./...` and independent build passed. Root `go work edit -json`, `go list -m`, service-path run, and `go -C services/matching-engine` fmt/vet/test/race/cover/build passed; built binary ran successfully. | Original root go.mod preserved unchanged and excluded from workspace; no existing Go source. No Git repository, so no commit hash. Initial checks failed on sandbox cache access and were rerun with approved access. Root path-based fmt failed; documented and verified `go -C` commands instead. Only bootstrap implemented; M1 awaits review. |
| M1 Domain model | NOT STARTED | — | |
| M2 Order book / engine | NOT STARTED | — | |
| M3 Journal / recovery | NOT STARTED | — | |
| M4 Gateway / risk / ledger | NOT STARTED | — | |
| M5 Kafka / market data / UI | NOT STARTED | — | |
| M6 Performance / operations | NOT STARTED | — | |
| M7 Advanced market behavior | NOT STARTED | — | Optional |

## 9. First prompt to paste into VS Code Codex

> Read `AGENTS.md` fully and inspect the existing repository without modifying files first. Show me the current tree, Git status, modules, and any code that would be affected by moving from the root `go.mod` to a matching-engine module. Then implement **only M0** safely, preserving existing files. Explain each change briefly in Thai, run relevant Go tests/build commands, update the M0 checklist and progress log based only on real evidence, and stop for my review before starting M1.

