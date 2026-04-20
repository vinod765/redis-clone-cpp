# redis-clone-cpp

> A from-scratch Redis-compatible server in C++ — built to learn, built to last.

This is not a toy. It implements a real subset of the Redis protocol, a custom hash table, an event-driven TCP server, and RDB persistence — the same core ideas that power one of the most widely used databases in the world.

---

## Architecture

```
Client (redis-cli / any RESP client)
        │
        │  TCP (port 6379)
        ▼
┌───────────────────────────────────┐
│         TCP Server                │
│   accept() → register with epoll  │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│         Event Loop                │
│   epoll_wait() → dispatch reads   │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│         RESP Parser               │
│   raw bytes → Command struct      │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│       Command Registry            │
│   lookup handler → execute        │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│            Store                  │
│   Hash Table + Expiry + Types     │
└──────────────┬────────────────────┘
               │
               ▼
┌───────────────────────────────────┐
│         RDB Persistence           │
│   periodic snapshot to disk       │
└───────────────────────────────────┘
```

---

## Stack

| Layer | Technology |
|---|---|
| Language | C++17 |
| Build system | CMake + Conan |
| I/O multiplexing | epoll (Linux) |
| Protocol | RESP2 (Redis Serialization Protocol) |
| Data structures | Custom hash table (open addressing) |
| Persistence | Simplified RDB format |
| Testing | Google Test |
| Benchmarking | Custom bench suite |

---

## Commands Implemented

### String Commands
| Command | Syntax | Status |
|---|---|---|
| SET | `SET key value [EX seconds]` | 🚧 |
| GET | `GET key` | 🚧 |
| DEL | `DEL key [key ...]` | 🚧 |
| EXISTS | `EXISTS key` | 🚧 |
| INCR | `INCR key` | 🚧 |
| DECR | `DECR key` | 🚧 |
| APPEND | `APPEND key value` | 🚧 |
| MSET | `MSET key value [key value ...]` | 🚧 |
| MGET | `MGET key [key ...]` | 🚧 |

### Generic Commands
| Command | Syntax | Status |
|---|---|---|
| EXPIRE | `EXPIRE key seconds` | 🚧 |
| TTL | `TTL key` | 🚧 |
| TYPE | `TYPE key` | 🚧 |
| KEYS | `KEYS pattern` | 🚧 |
| FLUSHALL | `FLUSHALL` | 🚧 |
| PING | `PING` | 🚧 |

### List Commands
| Command | Syntax | Status |
|---|---|---|
| LPUSH | `LPUSH key value [value ...]` | 🚧 |
| RPUSH | `RPUSH key value [value ...]` | 🚧 |
| LPOP | `LPOP key` | 🚧 |
| RPOP | `RPOP key` | 🚧 |
| LRANGE | `LRANGE key start stop` | 🚧 |
| LLEN | `LLEN key` | 🚧 |

### Hash Commands
| Command | Syntax | Status |
|---|---|---|
| HSET | `HSET key field value` | 🚧 |
| HGET | `HGET key field` | 🚧 |
| HDEL | `HDEL key field` | 🚧 |
| HGETALL | `HGETALL key` | 🚧 |
| HEXISTS | `HEXISTS key field` | 🚧 |

---

## Project Structure

```
redis-clone-cpp/
├── CMakeLists.txt
├── conanfile.txt
├── include/                  # all headers
│   ├── command_registry.h    # maps command names to handlers
│   ├── event_loop.h          # epoll wrapper
│   ├── expiry.h              # TTL tracking
│   ├── hash_table.h          # core data structure
│   ├── rdb.h                 # persistence
│   ├── resp_parser.h         # protocol parser
│   ├── resp_writer.h         # protocol serializer
│   ├── store.h               # main data store
│   └── tcp_server.h          # connection handling
├── src/
│   ├── main.cpp
│   ├── commands/
│   │   ├── command_registry.cpp
│   │   ├── generic_cmds.cpp
│   │   ├── hash_cmds.cpp
│   │   ├── list_cmds.cpp
│   │   └── string_cmds.cpp
│   ├── net/
│   │   ├── event_loop.cpp    # epoll_create, epoll_wait loop
│   │   └── tcp_server.cpp    # socket, bind, listen, accept
│   ├── persistence/
│   │   └── rdb.cpp           # save/load snapshots
│   ├── protocol/
│   │   ├── resp_parser.cpp   # parse RESP2 wire format
│   │   └── resp_writer.cpp   # serialize responses
│   └── store/
│       ├── expiry.cpp        # lazy + active expiry
│       ├── hash_table.cpp    # open addressing, robin hood hashing
│       └── store.cpp         # top-level store API
├── tests/
│   ├── test_commands.cpp
│   ├── test_expiry.cpp
│   ├── test_hash_table.cpp
│   └── test_resp_parser.cpp
└── bench/
    └── bench_main.cpp
```

---

## Building

```bash
# Install dependencies
conan install . --build=missing

# Build
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build

# Run
./build/redis-clone
```

## Testing

```bash
cmake --build build --target test
# or
cd build && ctest --verbose
```

## Connect with redis-cli

```bash
redis-cli -p 6379
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> SET name ironman
OK
127.0.0.1:6379> GET name
"ironman"
```

---

## Status

🚧 **Active development** — see [TODO.md](./TODO.md) for the full roadmap.

---

## License

MIT