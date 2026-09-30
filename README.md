# Cascade

A distributed, in-memory key-value store built from scratch in C# and .NET, a Redis-style cache with a custom binary wire protocol, LRU/TTL eviction, write-ahead-log persistence, and leader-follower replication.

Cascade is not a wrapper around an existing cache library. It is the server itself: a TCP listener that speaks a hand-rolled binary protocol, an in-memory store with real eviction and expiry semantics, a crash-safe persistence layer, and a replication mechanism that lets a follower take over if the leader goes down.

> Store data in memory. Survive a crash. Replicate to a follower. Understand every layer in between.

---

## Table of Contents

- [Why This Exists](#why-this-exists)
- [Features](#features)
- [Architecture](#architecture)
- [Wire Protocol](#wire-protocol)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [How It Works](#how-it-works)
- [Persistence](#persistence)
- [Replication](#replication)
- [Non-Goals](#non-goals-what-this-deliberately-does-not-do)
- [Roadmap](#roadmap)
- [Design Principles](#design-principles)
- [Development Setup](#development-setup)
- [Testing](#testing)
- [Benchmarking](#benchmarking)
- [License](#license)
- [Author](#author)
- [Status](#status)

---

## Why This Exists

Every high-traffic system eventually needs a cache layer in front of its database, but a cache layer that's actually worth trusting has to answer four hard questions:

1. What happens when memory fills up? → **eviction policy**
2. What happens when the process crashes? → **durability**
3. What happens when one server isn't enough? → **replication**
4. What does "the write succeeded" actually mean when there's more than one copy of the data? → **consistency**

Cascade exists to answer those questions in code, not just in theory — and to do it in idiomatic, high-performance C#, using the same low-level tools (raw sockets, zero-copy buffer parsing, hand-rolled protocols) that real database and cache engines are built with.

---

## Features

### Core Store
- In-memory key-value storage with byte-string keys and values.
- O(1) get/set/delete.
- LRU eviction under a configurable memory/entry-count cap.
- Per-key TTL with both lazy (on-access) and active (background sweep) expiry.

### Networking
- Custom binary wire protocol over raw TCP (no HTTP, no JSON on the hot path).
- Non-blocking, high-concurrency connection handling.
- Zero-copy (or close to it) request parsing using `Span<T>` / `Memory<T>`.

### Persistence
- Append-only write-ahead log (WAL) — every mutation is durable before it's acknowledged.
- Periodic full snapshotting, so the WAL doesn't grow unbounded.
- Crash recovery: replay snapshot + WAL tail on startup.

### Replication
- Single leader, one or more followers.
- Leader streams its WAL entries to followers over a dedicated connection.
- Manual promotion of a follower to leader (no automated failover — see [Non-Goals](#non-goals-what-this-deliberately-does-not-do)).

### Observability
- Structured logging of connection lifecycle, replication lag, and WAL/snapshot events.
- Basic runtime stats exposed over the protocol (key count, memory usage, uptime, replication offset).

### Client
- A minimal CLI client for interactive use (`cascade-cli GET foo`).
- A thin client library other .NET apps could reference directly.

---

## Architecture

```text
┌───────────────────────────────────────────┐
│                  Clients                   │
│         (cascade-cli / client library)     │
└───────────────────────┬─────────────────────┘
                        │ Cascade Wire Protocol (TCP)
                        ▼
┌───────────────────────────────────────────┐
│              Connection Layer              │
│                                             │
│   Socket accept loop                       │
│   Per-connection read/write pipeline       │
│   Protocol framing & parsing (Span/Memory) │
└───────────────────────┬─────────────────────┘
                        ▼
┌───────────────────────────────────────────┐
│                Command Layer               │
│                                             │
│   GET / SET / DEL / EXPIRE / PING / STATS  │
│   Command validation & dispatch            │
└───────────────────────┬─────────────────────┘
                        ▼
┌───────────────────────────────────────────┐
│                 Store Engine                │
│                                             │
│   Concurrent hash map                      │
│   LRU eviction                             │
│   TTL tracking (lazy + active sweep)       │
└──────────┬───────────────────┬─────────────┘
           ▼                   ▼
┌─────────────────────┐ ┌─────────────────────┐
│   Persistence Layer  │ │  Replication Layer  │
│                      │ │                      │
│   Write-ahead log    │ │  WAL streaming to    │
│   Snapshotting       │ │  follower(s)         │
│   Crash recovery     │ │  Replication offset  │
└─────────────────────┘ └─────────────────────┘
```

### Replication Topology

```text
┌─────────────┐        WAL stream        ┌─────────────┐
│   Leader     │ ───────────────────────▶ │  Follower    │
│  (accepts    │                          │ (read-only,  │
│   writes)    │ ◀─────────────────────── │  can be      │
└─────────────┘        ack / offset       │  promoted)   │
                                          └─────────────┘
```

---

## Wire Protocol

Cascade defines its own compact binary protocol rather than reusing HTTP or an existing serialization format. This is the part of the project that most directly exercises low-level C#.

### Request Frame

```text
┌─────────┬─────────┬────────────┬─────┬────────────┬───────┐
│ Magic   │ Opcode  │ Key Length │ Key │ Value Len  │ Value │
│ 1 byte  │ 1 byte  │ 4 bytes LE │ ... │ 4 bytes LE │  ...  │
└─────────┴─────────┴────────────┴─────┴────────────┴───────┘
```

- **Magic**: `0xCA` — identifies a Cascade frame, allows fast-fail on garbage input.
- **Opcode**: one of `GET (0x01)`, `SET (0x02)`, `DEL (0x03)`, `EXPIRE (0x04)`, `PING (0x05)`, `STATS (0x06)`, `REPLICATE (0x10)`.
- **Value Length / Value**: present only for `SET`; omitted for `GET`/`DEL`/`PING`.
- **TTL** (optional, `SET` only): an additional 8-byte signed integer (milliseconds; `-1` = no expiry) inserted after Value Length.

### Response Frame

```text
┌─────────┬──────────────┬───────┐
│ Status  │ Payload Len  │ Payload │
│ 1 byte  │ 4 bytes LE   │  ...    │
└─────────┴──────────────┴───────┘
```

- **Status**: `OK (0x00)`, `NOT_FOUND (0x01)`, `ERROR (0x02)`.

This protocol is intentionally simple — the point isn't protocol cleverness, it's demonstrating that you can parse and frame binary data correctly and efficiently over a raw socket.

---

## Technology Stack

| Component | Technology |
|---|---|
| Language | C# 13 |
| Runtime | .NET 10 |
| Networking | Raw `Socket` / `SocketAsyncEventArgs`, `System.IO.Pipelines` |
| Buffer handling | `Span<T>`, `Memory<T>`, `ArrayPool<T>` |
| Internal concurrency | `System.Threading.Channels`, `ConcurrentDictionary` |
| Persistence | Custom binary WAL + snapshot format, raw `FileStream` I/O |
| CLI client | `System.CommandLine` |
| Logging | `Microsoft.Extensions.Logging` + console/structured sink |
| Testing | xUnit |
| Benchmarking | BenchmarkDotNet, plus a custom load-generator client |
| Containerization | Docker (multi-stage build) |
| CI/CD | GitHub Actions |

> No ASP.NET Core, no Entity Framework, no JSON on the hot path — deliberately, to keep the project's value proposition about low-level systems work rather than framework usage.

---

## Project Structure

```text
Cascade/
│
├── src/
│   ├── Cascade.Core/
│   │   ├── Store/            # Hash map, LRU, TTL tracking
│   │   ├── Protocol/         # Frame parsing/encoding
│   │   └── Abstractions/
│   │
│   ├── Cascade.Server/
│   │   ├── Networking/       # Socket accept loop, connection handling
│   │   ├── Commands/         # Command dispatch
│   │   └── Program.cs
│   │
│   ├── Cascade.Persistence/
│   │   ├── Wal/              # Write-ahead log
│   │   └── Snapshots/
│   │
│   ├── Cascade.Replication/
│   │   ├── Leader/
│   │   └── Follower/
│   │
│   └── Cascade.Cli/
│       └── Program.cs
│
├── tests/
│   ├── Cascade.Core.Tests/
│   ├── Cascade.Persistence.Tests/
│   └── Cascade.Replication.Tests/
│
├── benchmarks/
│   └── Cascade.Benchmarks/
│
├── docs/
│   ├── protocol.md
│   ├── persistence.md
│   ├── replication.md
│   └── benchmarks.md
│
├── .github/
│   └── workflows/
│
├── .gitignore
├── LICENSE
├── README.md
└── Cascade.sln
```

---

## How It Works

1. Client opens a TCP connection to the server.
2. Client sends a request frame (e.g. `SET foo bar`).
3. Connection layer parses the frame using zero-copy buffer slicing.
4. Command layer validates and dispatches to the store engine.
5. Store engine applies the mutation in memory.
6. If the command is a mutation, it's appended to the WAL and, if replication is enabled, streamed to any connected followers — **before** the response is sent.
7. Server sends back a response frame.
8. In the background: TTL sweeper evicts expired keys; snapshotter periodically compacts the WAL.

---

## Persistence

- Every mutating command (`SET`, `DEL`, `EXPIRE`) is appended to an on-disk WAL before being acknowledged to the client — a crash between "applied in memory" and "written to disk" should never lose data.
- A background snapshotter periodically writes the full in-memory state to a snapshot file and truncates the WAL up to that point.
- On startup: load the most recent snapshot, then replay any WAL entries written after it.
- Deliberately simple: no compaction beyond periodic full snapshots, no incremental/partial snapshots.

---

## Replication

- One leader accepts all writes. Followers are read-only.
- The leader streams WAL entries to each connected follower in order, over a dedicated `REPLICATE` connection.
- Each follower tracks a replication offset and applies entries in order, giving eventual consistency between leader and follower.
- If the leader goes down, a follower can be **manually** promoted (an operator/CLI action) — there is no automatic leader election.

---

## Non-Goals (what this deliberately does not do)

Written down up front, on purpose, to keep the project from sprawling:

- No automated failover or leader election (Raft/Paxos-style consensus).
- No sharding/partitioning across multiple leaders.
- No multi-region replication.
- No pub/sub, transactions, or Lua scripting (Redis-style extras).
- No authentication/TLS on the wire protocol (documented as a known limitation, same approach as Apex).

These are explicitly out of scope for v1.0 rather than accidentally missing.

---

## Roadmap

### Phase 1 — Core Store & Protocol (MVP)
- [ ] Define and implement the wire protocol (framing, parsing, encoding).
- [ ] Implement the in-memory store (concurrent hash map).
- [ ] Implement LRU eviction under a configurable cap.
- [ ] Implement TTL (lazy expiry on access).
- [ ] Build the socket accept loop and per-connection handling.
- [ ] Build the CLI client.
- [ ] Basic unit tests for protocol parsing and store operations.

### Phase 2 — Durability
- [ ] Implement the write-ahead log.
- [ ] Implement snapshotting.
- [ ] Implement crash recovery (snapshot + WAL replay on startup).
- [ ] Active TTL sweep as a background task.
- [ ] Integration tests covering crash/restart scenarios.

### Phase 3 — Replication
- [ ] Implement the `REPLICATE` connection and WAL streaming.
- [ ] Implement follower-side apply logic and offset tracking.
- [ ] Implement manual promotion.
- [ ] Integration tests with a leader + follower pair.

### Phase 4 — Benchmarking & Polish
- [ ] BenchmarkDotNet micro-benchmarks (protocol parsing, store operations).
- [ ] Custom load-generator client for throughput/latency numbers under concurrency.
- [ ] Comparative notes against real Redis (context, not a competition).
- [ ] Docker packaging.
- [ ] GitHub Actions CI (build, test, benchmark on PR).
- [ ] Full docs pass (`protocol.md`, `persistence.md`, `replication.md`, `benchmarks.md`).
- [ ] Tagged v1.0.0 release.

---

## Design Principles

### Correct Before Fast
Durability and correctness guarantees come before raw throughput numbers.

### Explainable
Every design decision (eviction policy, consistency model, what's deferred) should be written down and defensible, not just implemented.

### Local-First
No cloud dependency required to build, run, or test the core system.

### Honest About Limitations
Non-goals are documented explicitly rather than left as silent gaps.

### Testable
Store, persistence, and replication logic are all independently testable without a live network stack.

---

## Development Setup

### Prerequisites

- .NET 10 SDK
- Git

### Clone and Scaffold

```bash
git clone https://github.com/YOUR_USERNAME/Cascade.git
cd Cascade
```

### Build

```bash
dotnet restore
dotnet build
```

### Run Tests

```bash
dotnet test
```

### Run the Server

```bash
dotnet run --project src/Cascade.Server
```

### Run the CLI Client

```bash
dotnet run --project src/Cascade.Cli -- SET foo bar
dotnet run --project src/Cascade.Cli -- GET foo
```

> Setup instructions may change as the architecture evolves.

---

## Testing

Testing areas include:

- Wire protocol framing and parsing (including malformed/partial frames).
- Store operations (get/set/delete, LRU eviction, TTL expiry).
- WAL write/replay correctness.
- Snapshot + WAL crash-recovery scenarios.
- Replication: leader/follower offset tracking, out-of-order/duplicate delivery handling.
- Concurrent access correctness under load.

---

## Benchmarking

- BenchmarkDotNet for micro-benchmarks: protocol parsing cost, store operation latency.
- A custom load-generator client for end-to-end throughput/latency under concurrent connections (the .NET equivalent of Apex's `wrk` benchmarking).
- Results and methodology documented in `docs/benchmarks.md`, including comparative context against real Redis where relevant.

---

## License

This project is licensed under the MIT License.

See the [LICENSE](LICENSE) file for details.

---

## Author

Built by Keletso Monyamane.

---

## Status

**Early Development**

Cascade is currently being designed and implemented, starting with Phase 1.
