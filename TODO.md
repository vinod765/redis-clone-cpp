# redis-clone-cpp — Build Roadmap

Work through these phases in order. Each one builds directly on the last.
Do not skip ahead — the order is intentional.

---

## Phase 0 — Build System
- [ ] CMakeLists.txt compiles a `main.cpp` that prints "redis-clone starting..."
- [ ] Conan set up — `conan install` works cleanly
- [ ] Google Test linked and a dummy test passes (`ctest` runs green)
- [ ] `.clang-format` added — consistent style from day one
- [ ] Verify build works: `cmake -B build && cmake --build build`

---

## Phase 1 — TCP Server
- [ ] Create a TCP socket on port 6379
- [ ] `setsockopt(SO_REUSEADDR)` — so restart doesn't fail with "address in use"
- [ ] `bind()` + `listen()`
- [ ] `accept()` loop — print "client connected" for each connection
- [ ] Set accepted sockets to **non-blocking** with `fcntl`
- [ ] Graceful shutdown on `SIGTERM` / `Ctrl+C`
- [ ] Test: `nc localhost 6379` connects successfully

---

## Phase 2 — Event Loop (epoll)
- [ ] `epoll_create1()` — create the epoll instance
- [ ] Register server fd with `EPOLLIN`
- [ ] `epoll_wait()` loop — dispatch on fd type
- [ ] On server fd ready → `accept()` → register new client fd
- [ ] On client fd ready → `read()` into per-client buffer
- [ ] Handle client disconnect (`read()` returns 0) → close + deregister
- [ ] Handle `EAGAIN` / `EWOULDBLOCK` correctly (non-blocking reads)
- [ ] Test: multiple `nc` clients connected simultaneously

---

## Phase 3 — RESP Parser
- [ ] Parse simple inline commands first: `PING\r\n` → Command{"PING", []}
- [ ] Parse RESP arrays: `*1\r\n$4\r\nPING\r\n`
- [ ] Parse multi-arg commands: `SET key value`
- [ ] Handle **partial reads** — buffer incomplete frames, wait for more data
- [ ] Return parse error on malformed input
- [ ] Unit tests for:
  - [ ] PING
  - [ ] SET with 2 args
  - [ ] MSET with multiple pairs
  - [ ] Partial frame buffering
  - [ ] Malformed input

---

## Phase 4 — RESP Writer
- [ ] `write_simple_string("+OK\r\n")`
- [ ] `write_error("-ERR message\r\n")`
- [ ] `write_integer(":42\r\n")`
- [ ] `write_bulk_string("$6\r\nfoobar\r\n")`
- [ ] `write_null_bulk("$-1\r\n")` — for missing keys
- [ ] `write_array(...)` — for KEYS, MGET, LRANGE responses
- [ ] Unit tests for each response type

---

## Phase 5 — PING + Command Registry
- [ ] `CommandRegistry` class with `register()` and `dispatch()`
- [ ] Implement `PING` → replies `+PONG\r\n`
- [ ] Unknown command → `-ERR unknown command\r\n`
- [ ] Wire everything together: epoll → parser → registry → writer → client
- [ ] **Milestone**: `redis-cli PING` returns `PONG` ✅

---

## Phase 6 — Hash Table
- [ ] Fixed-size bucket array with open addressing
- [ ] FNV-1a hash function for string keys
- [ ] Robin Hood insertion (track probe distance per slot)
- [ ] `insert(key, value)`
- [ ] `lookup(key)` → returns pointer or null
- [ ] `remove(key)`
- [ ] Resize (double capacity) when load factor > 0.75
- [ ] Unit tests:
  - [ ] Basic insert + lookup
  - [ ] Collision handling
  - [ ] Resize triggers correctly
  - [ ] Remove + re-insert

---

## Phase 7 — Store + String Commands
- [ ] `Store` class wrapping the hash table
- [ ] `RedisValue` variant type (string for now, extend later)
- [ ] Implement and register:
  - [ ] `SET key value`
  - [ ] `GET key` (returns null bulk if missing)
  - [ ] `DEL key [key ...]`
  - [ ] `EXISTS key`
  - [ ] `PING`
  - [ ] `FLUSHALL`
- [ ] **Milestone**: `redis-cli SET name ironman` + `GET name` works ✅

---

## Phase 8 — Expiry
- [ ] `ExpiryMap` — maps key → expiry timestamp (ms)
- [ ] `SET key value EX seconds` stores TTL
- [ ] `EXPIRE key seconds` sets TTL on existing key
- [ ] `TTL key` returns remaining time (-1 if no TTL, -2 if missing)
- [ ] Lazy expiry — check on every `get()`
- [ ] Active expiry — sweep 20 random TTL keys every 100ms in event loop
- [ ] Unit tests for lazy and active expiry

---

## Phase 9 — More String Commands
- [ ] `INCR key` — increment integer value (error if not integer)
- [ ] `DECR key`
- [ ] `APPEND key value`
- [ ] `MSET key value [key value ...]`
- [ ] `MGET key [key ...]`
- [ ] `TYPE key` → "string" / "list" / "hash" / "none"
- [ ] `KEYS *` → all keys (simple glob support optional)

---

## Phase 10 — List Commands
- [ ] Extend `RedisValue` to hold `std::list<std::string>`
- [ ] `LPUSH key value [value ...]`
- [ ] `RPUSH key value [value ...]`
- [ ] `LPOP key`
- [ ] `RPOP key`
- [ ] `LRANGE key start stop`
- [ ] `LLEN key`
- [ ] Type safety — error if key holds wrong type

---

## Phase 11 — Hash Commands
- [ ] Extend `RedisValue` to hold `std::unordered_map<string, string>`
- [ ] `HSET key field value`
- [ ] `HGET key field`
- [ ] `HDEL key field`
- [ ] `HGETALL key`
- [ ] `HEXISTS key field`
- [ ] Type safety checks

---

## Phase 12 — RDB Persistence
- [ ] `RDB::save(store, filename)` — write snapshot to disk
- [ ] Magic header: `REDIS0001`
- [ ] Serialize each key: type byte + key + value + optional expiry
- [ ] CRC64 checksum at end of file
- [ ] `RDB::load(filename)` — restore store from snapshot
- [ ] Auto-save: every 900s if 1+ changes, every 300s if 10+ changes
- [ ] Save on clean shutdown
- [ ] Unit tests: save → load → verify all keys match

---

## Phase 13 — Benchmarks
- [ ] Benchmark `hash_table` vs `std::unordered_map` (SET throughput)
- [ ] Benchmark RESP parser (commands/sec)
- [ ] Benchmark full server with `redis-benchmark` tool:
  ```bash
  redis-benchmark -p 6379 -t set,get -n 100000
  ```
- [ ] Document results in `bench/RESULTS.md`

---

## Phase 14 — Polish
- [ ] Config file support (`redis-clone.conf`)
  - [ ] port
  - [ ] save intervals
  - [ ] maxmemory
- [ ] Proper logging (levels: DEBUG, INFO, WARN, ERROR)
- [ ] `INFO` command — server stats
- [ ] `DBSIZE` command
- [ ] README screenshots / demo

---

## Backlog (stretch goals)
- [ ] `SUBSCRIBE` / `PUBLISH` (pub/sub)
- [ ] `MULTI` / `EXEC` (transactions)
- [ ] AOF (Append Only File) persistence
- [ ] Replication (leader/follower)
- [ ] Cluster mode (very hard — stretch)