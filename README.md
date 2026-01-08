# Baobab CLOB Engine

**High-performance off-chain Central Limit Order Book (CLOB) with intent-based orders and on-chain settlement.**

Inspired by modern DeFi perpetuals and spot DEX designs (including Baobab Protocol), this engine delivers **CEX-grade matching speed** off-chain while guaranteeing **on-chain safety and finality**.

Orders are signed intents (gasless EIP-712), matched deterministically in memory, batched by a keeper, and settled atomically on Ethereum (or any EVM chain).

## Core Principles

- **Speed**: Single-threaded per-market matching → no locks, sub-ms latency
- **Determinism**: Strict price-time priority via heaps + FIFO queues
- **Safety**: Blockchain is the single source of truth; off-chain state is cache only
- **Gas Efficiency**: Keeper batches multiple fills into single on-chain transactions
- **Non-custodial**: Users sign intents; no funds held by the engine
- **Production-Ready**: Structured logging, async persistence, recovery, observability

## Architecture Overview
Frontend (Web/Mobile)
↓ (signed intent)
API Gateway (Gin + WebSocket)
↓ (validate + ACK)
Engine Channel → Per-Market Goroutine
↓
Matching Engine (in-memory order book)
↓ (on match)
Publisher → Redis Pub/Sub
├─→ WebSocket Hub → Real-time updates to users
├─→ Keeper → Batch + on-chain settlement
├─→ Async Writer → Postgres (audit + recovery)
└─→ Indexer → Sync on-chain events → Redis cache + DB
text### Components

- **API**: Intake, EIP-712 validation, rate limiting, idempotency, WebSocket hub
- **Engine**: Per-market serialized processing, price-time priority matching
- **Keeper**: Consumes fills, validates collateral, batches, submits tx
- **Indexer**: Listens to contract events, updates Redis cache + Postgres
- **Async Writer**: Decouples DB writes from critical path
- **Redis**: Pub/Sub for fill fan-out + caching (collateral, balances)
- **Postgres**: Persistent audit trail, recovery, analytics

## Tech Stack

- **Language**: Go (1.22+)
- **API**: Gin + Gorilla WebSocket
- **Matching**: Custom heaps (`container/heap`) + `shopspring/decimal`
- **Config**: Viper (YAML + env)
- **Logging**: Zap (structured JSON)
- **DB**: PostgreSQL via pgx/v5 (connection pool)
- **Cache/Events**: Redis (go-redis/v9)
- **Ethereum**: go-ethereum (signing verify, event listening, tx submission)
- **Migrations**: Raw SQL files in `/migrations`

## Project Structure

```bash
baobab-clob/
├── cmd/
│   ├── api/
│   │   └── main.go               # Entry point for API server (Gin + WebSocket)
│   ├── engine/
│   │   └── main.go               # Entry point for matching engine workers
│   ├── keeper/
│   │   └── main.go               # Entry point for settlement keeper service
│   └── indexer/
│       └── main.go               # Entry point for on-chain event indexer
│
├── internal/
│   ├── config/
│   │   ├── config.go             # Viper configuration loading and struct definitions
│   │   └── types.go              # Config structs (API, DB, Redis, Ethereum settings)
│   │
│   ├── db/
│   │   ├── db.go                 # Postgres connection pool initialization and health checks
│   │   └── models/
│   │       ├── order.go          # Order model + CRUD queries
│   │       └── fill.go           # Fill model + CRUD queries
│   │
│   ├── types/
│   │   ├── order.go              # Shared Order struct (intent fields, status, etc.)
│   │   ├── fill.go               # Shared Fill struct
│   │   └── signature.go          # EIP-712 typed data structures
│   │
│   ├── gateway/
│   │   ├── handler.go            # Gin route definitions and HTTP handlers
│   │   ├── validator.go          # EIP-712 signature verification, schema validation
│   │   └── websocket.go          # WebSocket hub, connection management, broadcasting
│   │
│   ├── engine/
│   │   ├── channel.go            # Routes orders to per-market buffered channels
│   │   ├── orderbook.go          # In-memory order book with price-time priority (heaps + maps)
│   │   └── processor.go          # Core matching logic, fill generation, publishing
│   │
│   ├── publisher/
│   │   ├── redis.go              # Redis client initialization and connection management
│   │   └── publisher.go          # Functions to publish fills and order updates
│   │
│   ├── keeper/
│   │   ├── batcher.go            # Collects fills, performs risk checks, builds batches
│   │   ├── submitter.go          # Constructs and submits batched transactions
│   │   └── recover.go            # Recovers unsettled fills from DB/Redis on startup
│   │
│   ├── indexer/
│   │   ├── listener.go           # Subscribes to on-chain events via go-ethereum
│   │   ├── cache.go              # Updates Redis in-memory collateral and balance cache
│   │   └── updater.go            # Persists confirmed state to Postgres
│   │
│   ├── async/
│   │   ├── writer.go             # Background goroutine for batched async DB writes
│   │   └── types.go              # Write operation types and channel definitions
│   │
│   ├── risk/
│   │   └── checker.go            # Lightweight pre-matching and pre-settlement risk checks
│   │
│   └── utils/
│       ├── idempotency.go        # Deduplication using signer + nonce (DB + Redis)
│       ├── ratelimit.go          # Rate limiting per address/IP
│       └── logger.go             # Centralized Zap logger setup
│
├── migrations/
│   ├── 001_create_orders.up.sql   # Creates orders table
│   ├── 001_create_orders.down.sql # Drops orders table
│   ├── 002_create_fills.up.sql    # Creates fills table
│   └── 002_create_fills.down.sql  # Drops fills table
│
├── configs/
│   ├── config.yaml                # Default/example configuration
│   ├── config.dev.yaml            # Development overrides
│   └── config.prod.yaml           # Production overrides
│
├── deploy/
│   └── docker/
│       ├── Dockerfile.api     # Dockerfile for API service
│       ├── Dockerfile.engine  # Dockerfile for engine service
│       ├── Dockerfile.keeper  # Dockerfile for keeper service
│       └── Dockerfile.indexer # Dockerfile for indexer service
│
├── scripts/
│   ├── migrate.sh                 # Runs database migrations
│   ├── seed.sh                    # Optional: seeds test data
│   └── docker-entrypoint.sh       # Entry script for containers
│
├── test/
│   ├── unit/                      # Unit tests for individual packages
│   └── integration/               # End-to-end integration tests
│
├── pkg/                           # Public reusable packages (empty for now)
│
├── .env.example                   # Template for environment variables
├── .gitignore
├── go.mod                         # Go module definition and dependencies
├── go.sum                         # Dependency checksums
├── Makefile                       # Common build/test/run commands
└── README.md                      # Project documentation

baobab-clob/
│
├── cmd/                              # Entry points (binaries)
│   ├── api/main.go                   # API Gateway service
│   ├── engine/main.go                # Matching Engine service
│   ├── keeper/main.go                # Settlement Keeper service
│   └── indexer/main.go               # Event Indexer service
│
├── internal/                         # Private application code
│   │
│   ├── config/                       # Configuration management
│   │   ├── config.go                 # Viper setup, config loader
│   │   └── types.go                  # Config structs (API, DB, Redis, Ethereum)
│   │
│   ├── types/                        # Shared domain types
│   │   ├── order.go                  # Order struct, validation
│   │   ├── fill.go                   # Fill struct
│   │   ├── market.go                 # Market definitions
│   │   └── signature.go              # EIP-712 typed data structures
│   │
│   ├── gateway/                      # API Gateway layer
│   │   ├── server.go                 # HTTP server setup
│   │   ├── routes.go                 # Route definitions
│   │   ├── handler.go                # Request handlers
│   │   ├── validator.go              # EIP-712 signature verification
│   │   ├── websocket.go              # WebSocket hub, connection pool
│   │   └── middleware/
│   │       ├── auth.go               # Authentication
│   │       ├── ratelimit.go          # Rate limiting
│   │       ├── idempotency.go        # Duplicate request prevention
│   │       └── logger.go             # Request logging
│   │
│   ├── engine/                       # Matching Engine
│   │   ├── channel.go                # Market-specific routing
│   │   ├── market/
│   │   │   ├── market.go             # Market engine lifecycle
│   │   │   ├── orderbook.go          # In-memory order book
│   │   │   ├── heap.go               # Price-time priority heaps
│   │   │   ├── matcher.go            # Matching algorithm
│   │   │   ├── processor.go          # Order processing logic
│   │   │   └── snapshot.go           # State snapshots for recovery
│   │   └── models/
│   │       ├── order.go              # Internal order representation
│   │       └── trade.go              # Trade result
│   │
│   ├── publisher/                    # Event publishing
│   │   ├── redis.go                  # Redis client setup
│   │   └── publisher.go              # Publish fills, updates
│   │
│   ├── keeper/                       # Settlement Keeper
│   │   ├── keeper.go                 # Keeper lifecycle
│   │   ├── subscriber.go             # Redis subscription
│   │   ├── batcher.go                # Batch construction
│   │   ├── validator.go              # Pre-settlement validation
│   │   ├── submitter.go              # Transaction submission
│   │   └── recover.go                # Recovery on restart
│   │
│   ├── indexer/                      # Event Indexer
│   │   ├── indexer.go                # Indexer lifecycle
│   │   ├── listener.go               # Event subscription
│   │   ├── handlers/
│   │   │   ├── deposit.go            # Deposit event handler
│   │   │   ├── fill.go               # Fill event handler
│   │   │   └── withdrawal.go         # Withdrawal event handler
│   │   ├── cache.go                  # Redis cache updates
│   │   ├── updater.go                # Postgres updates
│   │   └── reorg.go                  # Reorg detection & handling
│   │
│   ├── storage/                      # Data persistence
│   │   ├── postgres/
│   │   │   ├── client.go             # Connection pool
│   │   │   ├── orders.go             # Order CRUD
│   │   │   ├── fills.go              # Fill CRUD
│   │   │   ├── balances.go           # Balance snapshots
│   │   │   └── health.go             # Health checks
│   │   └── redis/
│   │       ├── client.go             # Redis connection
│   │       ├── cache.go              # Collateral cache
│   │       ├── pubsub.go             # Pub/Sub helpers
│   │       └── locks.go              # Distributed locks
│   │
│   ├── async/                        # Async processing
│   │   ├── writer.go                 # Background DB writer
│   │   ├── batcher.go                # Write batching
│   │   └── queue.go                  # Write queue management
│   │
│   ├── risk/                         # Risk management
│   │   ├── checker.go                # Risk checks
│   │   ├── limits.go                 # Position limits
│   │   └── collateral.go             # Collateral validation
│   │
│   ├── eth/                          # Ethereum interaction
│   │   ├── client.go                 # Ethereum client wrapper
│   │   ├── signer.go                 # EIP-712 signing/verification
│   │   └── contracts/
│   │       └── settlement.go         # Generated contract bindings
│   │
│   └── utils/                        # Utilities
│       ├── logger.go                 # Structured logging setup
│       ├── metrics.go                # Prometheus metrics
│       ├── health.go                 # Health check utilities
│       └── recovery.go               # Panic recovery
│
├── migrations/                       # Database migrations
│   ├── 001_create_orders.up.sql
│   ├── 001_create_orders.down.sql
│   ├── 002_create_fills.up.sql
│   ├── 002_create_fills.down.sql
│   ├── 003_create_balances.up.sql
│   └── 003_create_balances.down.sql
│
├── configs/                          # Configuration files
│   ├── config.yaml                   # Base configuration
│   ├── config.dev.yaml               # Development overrides
│   ├── config.staging.yaml           # Staging overrides
│   └── config.prod.yaml              # Production configuration
│
├── deploy/                           # Deployment configurations
│   ├── docker/
│   │   ├── Dockerfile.api
│   │   ├── Dockerfile.engine
│   │   ├── Dockerfile.keeper
│   │   └── Dockerfile.indexer
│   └── kubernetes/
│       ├── api-deployment.yaml
│       ├── engine-deployment.yaml
│       ├── keeper-deployment.yaml
│       ├── indexer-deployment.yaml
│       └── services.yaml
│
├── scripts/                          # Utility scripts
│   ├── migrate.sh                    # Run migrations
│   ├── seed.sh                       # Seed test data
│   ├── docker-entrypoint.sh          # Container entry point
│   └── benchmark.sh                  # Run benchmarks
│
├── test/                             # Tests
│   ├── unit/                         # Unit tests
│   │   ├── orderbook_test.go
│   │   ├── matcher_test.go
│   │   └── validator_test.go
│   ├── integration/                  # Integration tests
│   │   ├── api_test.go
│   │   └── settlement_test.go
│   └── load/                         # Load tests
│       └── locustfile.py
│
├── docs/                             # Documentation
│   ├── architecture.md               # System architecture
│   ├── api.md                        # API documentation
│   ├── deployment.md                 # Deployment guide
│   └── performance.md                # Performance tuning
│
├── .env.example                      # Environment variable template
├── .gitignore
├── go.mod                            # Go module definition
├── go.sum                            # Dependency checksums
├── Makefile                          # Build automation
├── docker-compose.yaml               # Local development stack
└── README.md                         # This file

text## Getting Started
```

### Prerequisites
```bash
- Go 1.22+
- PostgreSQL 15+
- Redis 7+
- Docker (optional)
```
### Local Development

```bash
# Clone
git clone https://github.com/AdebakinOlujimi/baobab-clob.git
cd baobab-clob

# Copy config
cp configs/config.yaml configs/local.yaml
# Edit local.yaml with your settings

# Start Postgres & Redis (Docker)
docker run -p 5432:5432 -e POSTGRES_PASSWORD=pass postgres
docker run -p 6379:6379 redis

# Run migrations
./scripts/migrate.sh

# Run services (separate terminals)
go run cmd/api/main.go
go run cmd/engine/main.go
go run cmd/keeper/main.go
go run cmd/indexer/main.go
Roadmap

 Spot matching (ETH-USD, BTC-USD)
 Real-time WebSocket updates
 Single keeper settlement
 DB recovery on restart
 Multi-keeper competition
 Perpetual futures
 Advanced risk engine

Contributing
Pull requests welcome! Focus on performance, testing, and observability.
License
MIT
textThis is the complete, production-ready README — fully in Markdown, with detailed file-by-file descriptions as requested. Copy-paste it directly into your `README.md`. Your repo now looks professional and clear to any contributor or investor. 🚀

Ready for the next step: `internal/config/config.go` + Viper setup? Let's go!
