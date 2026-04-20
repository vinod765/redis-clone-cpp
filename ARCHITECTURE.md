# redis-clone-cpp — Architecture

## Core Design Philosophy

Single-threaded, event-driven, non-blocking — exactly like real Redis.

No threads. No locks. No async frameworks. Just epoll, a tight event loop, and fast data structures. This is the model that made Redis famous and it's what I am building here.

---

## The Event Loop Model

```
┌─────────────────────────────────────────────┐
│              Event Loop (epoll)             │
│                                             │
│  while (true) {                             │
│    events = epoll_wait(epfd, ...)           │
│                                             │
│    for each event:                          │
│      if fd == server_fd → accept()          │
│      if fd == client_fd → read() → parse()  │
│  }                                          │
└─────────────────────────────────────────────┘
```

All file descriptors are set to **non-blocking**. `epoll_wait` blocks until something is ready. This means one thread handles thousands of clients efficiently — no context switching, no lock contention.

---

## Component Breakdown

### 1. TCP Server (`net/tcp_server`)
Owns the listening socket.

```
socket(AF_INET, SOCK_STREAM)
  → setsockopt(SO_REUSEADDR)
  → bind(port 6379)
  → listen()
  → register with epoll (EPOLLIN)

on EPOLLIN:
  → accept() → new client fd
  → fcntl(fd, F_SETFL, O_NONBLOCK)
  → register client fd with epoll
```

### 2. Event Loop (`net/event_loop`)
The heartbeat of the server.

```cpp
// Core loop
while (running) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, timeout_ms);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == server_fd)
            handle_new_connection();
        else
            handle_client_data(events[i].data.fd);
    }
    // Also runs expiry sweep here
    check_expiry();
}
```

### 3. RESP Parser (`protocol/resp_parser`)
Converts raw TCP bytes into a `Command` struct.

RESP2 wire format:
```
*3\r\n          ← array of 3 elements
$3\r\n          ← bulk string, 3 bytes
SET\r\n
$4\r\n
name\r\n
$7\r\n
ironman\r\n
```

Parser output:
```cpp
struct Command {
    std::string name;               // "SET"
    std::vector<std::string> args;  // ["name", "ironman"]
};
```

Key challenge: **partial reads**. TCP can deliver data in chunks. The parser must handle incomplete RESP frames and buffer them until a full command arrives.

### 4. RESP Writer (`protocol/resp_writer`)
Serializes responses back to the client.

```cpp
// Response types
"+OK\r\n"                    // Simple string
"-ERR unknown command\r\n"   // Error
":42\r\n"                    // Integer
"$6\r\nfoobar\r\n"           // Bulk string
"*2\r\n$3\r\nfoo\r\n..."     // Array
"$-1\r\n"                    // Null bulk string (key not found)
```

### 5. Command Registry (`commands/command_registry`)
Maps command name strings to handler functions.

```cpp
using Handler = std::function<Response(Store&, const Command&)>;

std::unordered_map<std::string, Handler> registry = {
    {"SET",  handle_set},
    {"GET",  handle_get},
    {"DEL",  handle_del},
    // ...
};
```

Lookup is O(1). Adding new commands is just registering a new handler.

### 6. Store (`store/store`)
The main in-memory database. Sits on top of the hash table.

```cpp
class Store {
    HashTable<std::string, RedisValue> data;
    ExpiryMap expiry;

public:
    void set(const std::string& key, RedisValue val, int ttl_ms = -1);
    std::optional<RedisValue> get(const std::string& key);
    bool del(const std::string& key);
    bool exists(const std::string& key);
};
```

`RedisValue` is a variant type holding all supported value types:
```cpp
using RedisValue = std::variant<
    std::string,             // String
    std::list<std::string>,  // List
    std::unordered_map<std::string, std::string>  // Hash
>;
```

### 7. Hash Table (`store/hash_table`)
Custom implementation — no `std::unordered_map` for the top-level store.

- **Open addressing** with **Robin Hood hashing**
- Robin Hood reduces variance in probe lengths — elements that are "rich" (close to home) give up their slot to elements that are "poor" (far from home)
- Load factor kept below 0.75 — resize doubles capacity
- Keys are std::string, hashed with FNV-1a

```
Bucket:  [ key | value | probe_distance ]
Empty:   probe_distance = -1
```

### 8. Expiry (`store/expiry`)
Two-pronged expiry strategy — same as real Redis:

**Lazy expiry** — check on access:
```cpp
auto get(key) {
    if (is_expired(key)) { del(key); return null; }
    return data[key];
}
```

**Active expiry** — periodic sweep:
```
Every 100ms: sample 20 random keys with TTLs
             delete expired ones
             if > 25% were expired: repeat immediately
```

### 9. RDB Persistence (`persistence/rdb`)
Periodic snapshots to disk — simplified RDB format.

```
File format:
  [MAGIC: "REDIS0001"]
  [DB_SELECTOR: 0]
  for each key:
    [TYPE byte]
    [KEY length + bytes]
    [VALUE length + bytes]
    [EXPIRY ms, if set]
  [EOF marker]
  [CRC64 checksum]
```

Save triggered:
- Every N seconds if M keys changed (configurable)
- On SIGTERM / graceful shutdown

---

## Data Flow: Full Request Lifecycle

```
redis-cli sends: "SET name ironman\r\n" (as RESP)

1. epoll_wait() → client fd is readable
2. read(fd, buf) → raw bytes into client buffer
3. resp_parser → Command{ "SET", ["name", "ironman"] }
4. command_registry.lookup("SET") → handle_set()
5. handle_set() → store.set("name", "ironman")
6. store.set() → hash_table.insert(key, value)
7. handle_set() returns Response::ok()
8. resp_writer → "+OK\r\n"
9. write(fd, "+OK\r\n")
10. redis-cli receives: OK
```

---

## Key Learning Concepts by Component

| Component | Concept you learn |
|---|---|
| event_loop | epoll, non-blocking I/O, the C10K problem |
| tcp_server | BSD sockets, bind/listen/accept lifecycle |
| resp_parser | Protocol design, partial reads, state machines |
| hash_table | Open addressing, Robin Hood hashing, load factors |
| expiry | Lazy vs active strategies, time-based data |
| rdb | Binary file formats, serialization, checksums |
| store | Variant types, type dispatch in C++ |
| command_registry | Function pointers, dispatch tables |