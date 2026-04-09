# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run the service
go run main.go

# Build binary
go build -o mini-forge

# Run all tests
go test -v ./...

# Run tests for a specific package
go test -v -timeout 30s ./internal/api_handlers/

# Load testing (requires k6: brew install k6)
./run_test_and_analyze.sh
```

Configuration is loaded from `.env` (copy `.env.example`) and `config/config.yml`. Key env vars: `PORT`, `RANGE_SIZE`, `DEBUG_ENABLED`, `CONFIG_PATH`.

## Architecture

Mini-Forge is a **Key Generation Service (KGS)** — it serves short (7-char), Base62-encoded, unique keys via `GET /get-key`. Its core design solves the DB bottleneck problem by pre-generating keys into an in-memory pool.

### Request flow

```
GET /get-key → ApiHandler → KeyPool.Get() → buffered channel → return key
```

If the pool drops below 10% of `RANGE_SIZE`, a background goroutine (`refiller`) fetches a new batch:

```
refiller → RangeCounterRepository.GetAndIncrement() → SQLite (row-locked transaction)
         → generate Base62 keys for the reserved range → Put() into channel
```

### Key components


| Package                                 | Role                                                                   |
| --------------------------------------- | ---------------------------------------------------------------------- |
| `internal/services/key_pool.go`         | In-memory buffered channel + background refill goroutine               |
| `internal/repository/range_counter.go`  | Atomic range reservation via `SELECT FOR UPDATE` transaction           |
| `internal/api_handlers/api_handlers.go` | Single HTTP handler wiring `KeyPool` to chi router                     |
| `internal/utils/helpers.go`             | Bijective shuffle + Base62 encoding (makes keys appear non-sequential) |
| `internal/database/database.go`         | SQLite init and migrations via GORM                                    |
| `internal/utils/config.go`              | YAML + `.env` config loading                                           |


### Concurrency model

- Multiple service instances share one SQLite DB; transaction locking on `range_counters` (single row, `id=1`) ensures non-overlapping ranges.
- The pool is a Go channel (`chan string`); `KeyPool.Get()` blocks if empty.
- The refiller runs every 1 second and checks pool occupancy.

### Key capacity

Base62 with 7 characters → 62^7 ≈ 3.5 trillion unique keys.