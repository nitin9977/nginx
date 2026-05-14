# NGINX Architecture Overview

**Audience**: Engineers ramping up on nginx. Assumes C experience, some UNIX systems knowledge.
**Last Updated**: 2024
**Status**: Work in Progress (Principal Engineer Review)

---

## Executive Summary

NGINX is a high-performance event-driven web server that uses a **master-worker process model** with **shared event loops** running in isolated worker processes. The architecture prioritizes concurrency, minimal memory footprint, and non-blocking I/O using OS event multiplexing (epoll/kqueue/select).

**Key Design Principles:**
- **Single-threaded workers**: Each worker process runs its own event loop independently
- **Connection pooling**: Connections and buffers allocated from memory pools for predictable performance
- **Minimal shared state**: Workers rarely synchronize; accept mutex is the primary contention point
- **Zero-copy where possible**: Output chains and buffer chains enable scatter-gather I/O
- **Graceful degradation**: Timeouts, connection limits, and backpressure prevent resource exhaustion

---

## Module Dependency Tree

```
Master Process (ngx_master_process_cycle)
    │
    ├─→ Signal Handler (SIGHUP, SIGUSR1, SIGTERM, SIGQUIT, SIGCHLD)
    │
    ├─→ Worker 0   (ngx_worker_process_cycle)
    │    │
    │    └─→ Event Loop (ngx_process_events_and_timers)
    │         ├─→ Accept Mutex (ngx_accept_mutex, ngx_shmtx_t)
    │         ├─→ Event Multiplexor (epoll/kqueue/select)
    │         ├─→ Connection Handler
    │         ├─→ Timer Queue (ngx_event_timer_t)
    │         └─→ Posted Events Queue
    │
    ├─→ Worker N   (ngx_worker_process_cycle)
    │    └─→ Event Loop (identical to Worker 0)
    │
    └─→ Cache Manager Process (optional)

Configuration
    │
    ├─→ Cycle Initialization (ngx_init_cycle)
    │    ├─→ Configuration Parse (ngx_conf_parse)
    │    ├─→ Listening Sockets (ngx_listening_t)
    │    └─→ Module Initialization (ngx_event_process_init)
    │
    └─→ Modules
         ├─→ Core (nginx.c, ngx_cycle.c, ngx_connection.c)
         ├─→ Event (ngx_event.c, ngx_event_*.c)
         ├─→ HTTP (ngx_http_*.c)
         ├─→ Stream (optional)
         └─→ Mail (optional)
```

---

## Core Architecture Components

### 1. **Master Process** (`src/os/unix/ngx_process_cycle.c:74`)

**Responsibility**: Process management, signal coordination, graceful reload.

**Key Functions**:
- `ngx_master_process_cycle()` (line 74): Main master loop
- `ngx_start_worker_processes()` (line 336): Pre-fork N workers
- `ngx_signal_worker_processes()` (line 667): Broadcast signals to workers
- `ngx_reap_children()` (line 513): Reap zombie processes (SIGCHLD)

**Signal Handling** (line 56-82):
- **SIGHUP** (reconfigure): Reload configuration, spawn new workers
- **SIGUSR1** (reopen): Rotate log files
- **SIGUSR2** (noaccept): Gracefully disable new connections
- **SIGTERM** (terminate): Kill workers immediately
- **SIGQUIT** (quit): Graceful shutdown (wait for connections)
- **SIGCHLD**: Child process died, reap and respawn if needed

**Flow**:
```c
ngx_master_process_cycle(cycle)
  ├─ spawn N workers via ngx_start_worker_processes()
  ├─ register signal handlers
  └─ for (;;) {
       wait for signals (pause())
       if (ngx_reap) reap_children()
       if (ngx_reconfigure) reload config + spawn new workers
       if (ngx_terminate) exit immediately
       if (ngx_quit) graceful shutdown
     }
```

**Invariant**: Master is single-threaded and never processes requests. Its only job is coordination.

---

### 2. **Worker Process** (`src/os/unix/ngx_process_cycle.c:699`)

**Responsibility**: Accept connections, parse HTTP, run event loop, send responses.

**Initialization** (line 753: `ngx_worker_process_init()`):
- Drop privileges to `user` directive
- Set CPU affinity (if configured)
- Set resource limits (file descriptors, etc.)
- Install signal handlers (only a subset; most are ignored)
- Initialize per-worker state

**Main Loop** (line 710-748):
```c
for (;;) {
    if (ngx_exiting && no_timers_left()) exit;
    
    ngx_process_events_and_timers(cycle);  // ← Heart of the worker
    
    if (ngx_terminate) exit immediately;
    if (ngx_quit) graceful shutdown;
    if (ngx_reopen) rotate logs;
}
```

**Key Points**:
- All workers run **identical event loops** — no explicit load balancing
- Accept mutex (line 219-239 in `ngx_event.c`) ensures only one worker accepts per iteration
- Workers operate independently; no shared state except atomic counters and accept mutex

---

### 3. **Event Loop** (`src/event/ngx_event.c:195`)

**Function**: `ngx_process_events_and_timers(ngx_cycle_t *cycle)`

**Purpose**: The heartbeat of each worker. Handles I/O readiness, timers, and graceful shutdown.

**Algorithm**:

1. **Calculate timer** (line 205):
   - Find next pending timer via `ngx_event_find_timer()`
   - If no timers, block indefinitely (or cap at 500ms on Windows for signal safety)

2. **Try to acquire accept mutex** (line 224):
   - Only one worker should accept new connections per iteration
   - If lock held, post accepted events for later processing (line 229: `NGX_POST_EVENTS`)
   - If lock fails, reduce accept timer (line 235: `ngx_accept_mutex_delay`)

3. **Call `ngx_process_events()`** (line 248):
   - Invokes platform-specific multiplexor (epoll/kqueue/select)
   - Blocks for `timer` milliseconds waiting for I/O readiness
   - Populates event queues: `ngx_posted_accept_events`, `ngx_posted_events`

4. **Process accepted connections** (line 255):
   - Run callbacks for newly accepted sockets
   - Call `ngx_http_init_connection()` (HTTP) or protocol handler

5. **Unlock and expire timers** (line 257-261):
   - Release accept mutex
   - Call `ngx_event_expire_timers()` to fire timeouts

6. **Process remaining posted events** (line 263):
   - Run data handlers (read/write callbacks for established connections)

**Event Flags**:
- `NGX_UPDATE_TIME` (line 206): Recalculate current time after multiplexor
- `NGX_POST_EVENTS` (line 229): Defer event processing for fairness
- Timer calculations respect `timer_resolution` directive (for batching)

**Invariant**: The event loop never blocks indefinitely. Timers ensure progress even with no I/O.

---

### 4. **Connection Management** (`src/core/ngx_connection.c`, `src/core/ngx_connection.h:127`)

**Structure**: `ngx_connection_t` (127-206 lines)

**Key Fields**:
```c
struct ngx_connection_s {
    // I/O
    ngx_socket_t         fd;              // file descriptor
    ngx_event_t         *read, *write;    // read/write event objects
    ngx_recv_pt          recv;            // function ptr: recv implementation
    ngx_send_pt          send;            // function ptr: send implementation
    
    // Application Data
    void                *data;            // protocol-specific data (ngx_http_request_t)
    ngx_listening_t     *listening;       // listening socket reference
    ngx_pool_t          *pool;            // connection's memory pool
    ngx_buf_t           *buffer;          // read buffer
    
    // Metadata
    ngx_log_t           *log;             // logger
    ngx_atomic_uint_t    number;          // unique connection number
    
    // Flags (bit fields)
    unsigned buffered:1;                  // has unsent data
    unsigned log_error:1;                 // log errors
    unsigned timedout:1;                  // timeout occurred
    unsigned error:1;                     // error flag
    unsigned destroyed:1;                 // connection is destroyed
    unsigned close:1;                     // close after I/O
    unsigned sendfile:1;                  // can use sendfile
    unsigned tcp_nodelay:1;               // TCP_NODELAY enabled
    
    // SSL/TLS
    ngx_ssl_connection_t *ssl;            // OpenSSL connection
    
    // QUIC (experimental)
    ngx_quic_stream_t   *quic;            // QUIC stream ref
};
```

**Lifecycle**:
```
1. Socket Accept (listening socket readable)
   ├─ ngx_handle_accept_event() → ngx_event_accept() 
   ├─ accept(2) system call
   ├─ allocate ngx_connection_t from pool
   └─ call protocol handler: ngx_http_init_connection()
   
2. Request Processing
   ├─ parse HTTP request
   ├─ run request handler (locate content, filter chain)
   └─ send response
   
3. Close or Reuse
   ├─ if (keep-alive && reusable)
   │   └─ reset connection, wait for next request
   └─ else
       └─ ngx_close_connection()
           ├─ shutdown(2) socket
           ├─ unregister from epoll/kqueue
           └─ free connection from pool
```

**Memory Layout** (cache-line aware):
- Connection struct is pool-allocated (typically 512-4096 bytes per connection)
- Each connection has its own pool: `connection->pool`
- Buffers allocated from connection pool; freed en masse on close

---

### 5. **Memory Allocation** (`src/core/ngx_palloc.c`)

**Model**: Pool-based allocation for predictable deallocation.

**Structure**: `ngx_pool_t`
```c
struct ngx_pool_s {
    ngx_pool_data_t  d;                   // data block (free/end/next chain)
    size_t           max;                 // threshold for large allocations
    ngx_pool_t      *current;             // current block for allocation
    ngx_pool_large_t *large;              // separate list for large blocks
    ngx_pool_cleanup_t *cleanup;          // cleanup handlers
    ngx_log_t       *log;                 // logger
};
```

**Allocation Strategy** (line 123: `ngx_palloc()`):

1. **Small allocations** (< max, usually 3968 bytes):
   - Try to fit in current block's free space
   - If no space, allocate new block via `ngx_palloc_block()`
   - Blocks linked via `next` pointer (single-linked chain)

2. **Large allocations** (>= max):
   - Allocate separate memory block
   - Track via `large` linked list
   - Enables selective deallocation (important for some use cases)

**Deallocation** (line 47: `ngx_destroy_pool()`):

```c
ngx_destroy_pool(pool):
  ├─ call cleanup handlers (line 53-59)
  ├─ free all large blocks (line 83-87)
  └─ free all pool blocks (line 89-95)
     └─ single pass; O(N) where N = number of blocks
```

**Why Pools?**
- **Predictable**:  Avoid fragmentation; O(1) deallocation
- **Bulk cleanup**: Free 1000s of allocations with one call
- **Reduced syscalls**: Batch allocations into larger OS blocks
- **NUMA-aware**: Can align pools to NUMA nodes (future work)

**Critical Assumption** ⚠️: Pools assume all data within a scope (e.g., per-connection) is freed together. Leaks are rare but hard to debug.

---

### 6. **HTTP Request Parsing** (`src/http/ngx_http_parse.c:108`)

**Function**: `ngx_http_parse_request_line(ngx_http_request_t *r, ngx_buf_t *b)`

**Purpose**: State machine to parse HTTP request line: `METHOD URI HTTP/1.1\r\n`

**States** (line 111-138):
- `sw_start` → `sw_method` → `sw_spaces_before_uri` → `sw_schema` → `sw_uri` → `sw_http_H` → version parsing → `sw_almost_done`

**Character-by-Character Parsing**:
```c
for (p = b->pos; p < b->last; p++) {
    ch = *p;
    
    switch (state) {
    case sw_start:
        // Validate method char or transition
        // Update request_start, method, method_end
        
    case sw_uri:
        // Validate URI char, handle special cases (IPv6, ports, etc.)
        // Update uri, uri_start, uri_end
        
    case sw_http_H/T/T/P:
        // Validate HTTP version
        
    case sw_first_major_digit / sw_major_digit:
        // Parse HTTP major version (1-9)
        
    case sw_spaces_after_digit:
        // Transition to done
    }
}
```

**Return Values**:
- `NGX_OK` (1): Request line complete (state = `sw_almost_done`)
- `NGX_AGAIN` (-11): Need more data; continue parsing next buffer
- `NGX_HTTP_PARSE_INVALID_REQUEST` (400): Malformed input

**Invariants**:
- State machine is **resumable**: Can be called multiple times with new data
- Validates HTTP/1.0 and HTTP/1.1 only (HTTP/2, HTTP/3 handled separately)
- No lookahead; processes one byte at a time

**Security Concerns**:
- **URI length**: Validated by `large_client_header_buffers` limit (default 32KB)
- **Method validation**: Only alphanumeric; custom methods allowed
- **Version parsing**: Strict; invalid versions rejected

---

### 7. **Buffer Management** (`src/core/ngx_buf.h:20`)

**Structure**: `ngx_buf_t`
```c
struct ngx_buf_s {
    u_char    *pos;                  // current read position
    u_char    *last;                 // end of valid data
    off_t      file_pos, file_last;  // file positions (if in_file=1)
    u_char    *start, *end;          // buffer boundaries
    
    // Flags indicate buffer semantics
    unsigned temporary:1;            // data can be modified
    unsigned memory:1;               // read-only memory (don't modify)
    unsigned mmap:1;                 // mmap'd; don't modify
    unsigned recycled:1;             // can be reused
    unsigned in_file:1;              // data in file, not memory
    unsigned flush:1;                // flush partial output
    unsigned sync:1;                 // sync before next data
    unsigned last_buf:1;             // last buffer in final chunk
    unsigned last_in_chain:1;        // last buffer in current chain
};

struct ngx_chain_s {
    ngx_buf_t    *buf;               // buffer
    ngx_chain_t  *next;              // next in chain
};
```

**Allocation** (`src/core/ngx_buf.c`):
- `ngx_create_temp_buf(pool, size)`: Allocate buffer + chain from pool
- Buffers typically 4KB-16KB for initial read, sized by directive

**Zero-Copy Strategy**:
1. Read data into buffer
2. Create buffer chain with `in_file=1` (points to file, not memory)
3. Use sendfile(2) to transmit directly from file to socket
4. No copy: kernel reads file → sends to network

**Recycling**:
- Buffers marked `recycled=1` can be freed/reused by output chain filter
- Reduces allocation churn for high-throughput scenarios

---

### 8. **Synchronization & Concurrency** 

#### Accept Mutex (`src/event/ngx_event.c:51-58`)

**Problem**: Multiple workers trying to accept from the same listening socket causes thundering herd (all wake up, only one succeeds).

**Solution**: Shared mutex serializes accept.

**Implementation** (line 219-239):
```c
if (ngx_use_accept_mutex) {
    if (ngx_accept_disabled > 0) {
        ngx_accept_disabled--;          // backpressure: skip accept
    } else {
        if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
            return;                      // lock failed; try next iteration
        }
        if (ngx_accept_mutex_held) {
            flags |= NGX_POST_EVENTS;    // defer event processing
        } else {
            timer = ngx_accept_mutex_delay;  // short timeout to retry lock
        }
    }
}
```

**Fields**:
- `ngx_accept_mutex_ptr`: Pointer to shared memory (process-mapped)
- `ngx_accept_mutex`: Mutex control struct (spinlock + semaphore)
- `ngx_use_accept_mutex`: Config flag (multi-process only)
- `ngx_accept_disabled`: Backpressure counter (incremented per accept)

**Behavior**:
- Only worker holding mutex processes accept events
- Others skip accept events but still handle data (reads/writes)
- Backpressure: If `connection_count > max_clients * 0.75`, stop accepting

#### Atomic Operations (`src/os/unix/ngx_atomic.h`)

```c
ngx_atomic_cmp_set(lock, old, new);    // compare-and-swap
ngx_atomic_fetch_add(addr, value);     // atomic add
```

**Used For**:
- Connection counter (`ngx_connection_counter`)
- Statistics (accepted, handled, active, reading, writing, waiting)
- Accept mutex spinlock

#### No Fine-Grained Locking

**Design Decision**: Nginx avoids per-connection locks. Why?
- Connections are independent (no sharing)
- Event loop is single-threaded per worker
- No reader/writer contention within a worker

**Shared State** (protected by accept mutex):
- Listening socket's accept queue
- Global statistics (optional, for monitoring)

---

### 9. **Signal Handling** (`src/os/unix/ngx_process.c:24`)

**Signal Handler**: `ngx_signal_handler()` (static, line 24)

**Design**:
- Async-signal-safe only: set atomic flags, call `write()` to notify channel
- Never acquires locks, calls malloc, or runs long code

**Signal Disposition**:
```c
sigaction(NGX_RECONFIGURE_SIGNAL (SIGHUP))    → ngx_reconfigure = 1
sigaction(NGX_REOPEN_SIGNAL (SIGUSR1))        → ngx_reopen = 1
sigaction(NGX_NOACCEPT_SIGNAL (SIGUSR2))      → ngx_noaccept = 1
sigaction(NGX_TERMINATE_SIGNAL (SIGTERM))     → ngx_terminate = 1
sigaction(NGX_SHUTDOWN_SIGNAL (SIGQUIT))      → ngx_quit = 1
sigaction(SIGCHLD)                            → ngx_reap = 1
sigaction(SIGIO)                              → ngx_sigio = 1
sigaction(SIGALRM)                            → ngx_sigalrm = 1

// Ignored
sigaction(SIGPIPE)                            SIG_IGN
sigaction(SIGSYS)                             SIG_IGN
```

**Signal Safety**:
- Flags are `sig_atomic_t` (guaranteed atomic read/write)
- Master and worker respond to signals asynchronously in main loops
- No critical sections interrupted by signals

**Invariant**: All signal handlers are async-signal-safe per POSIX.1-2008.

---

## Data Flow: Request Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Socket Accept                                                │
│    listening socket → readable event                            │
│    → ngx_handle_accept_event()                                  │
│    → accept(2) + allocate ngx_connection_t                      │
│    → post to ngx_posted_accept_events queue                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. Connection Initialization (HTTP)                             │
│    ngx_http_init_connection()                                   │
│    → register read event (wait for HTTP request)                │
│    → set handler: ngx_http_wait_request_handler()              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Request Line Parsing                                         │
│    read event fires → ngx_http_process_request_line()          │
│    → recv(2) into connection->buffer                            │
│    → ngx_http_parse_request_line() state machine               │
│    → if complete: ngx_http_process_request_headers()           │
│    → else: loop, wait for more data                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Header Parsing                                               │
│    ngx_http_process_request_headers()                           │
│    → parse each header line (Content-Length, Host, etc.)        │
│    → accumulate in request struct                               │
│    → validate headers (size limits, etc.)                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. Request Processing (Handler Phase)                           │
│    ngx_http_handler()                                           │
│    → determine location (regex matching)                        │
│    → run location handler (static file, proxy, script, etc.)    │
│    → collect output via filter chain                            │
│    → apply transformations (gzip, SSI, etc.)                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. Response Output (Filter Chain)                               │
│    ngx_http_output_chain()                                      │
│    → wrap response in ngx_chain_t (buffer chain)                │
│    → call ngx_writev() or sendfile()                            │
│    → handle backpressure (partial writes)                       │
│    → register write event if buffer not empty                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. Keep-Alive / Close                                           │
│    if (keep-alive && connection valid):                         │
│       → ngx_reset_connection()                                  │
│       → register read event (wait for next request)             │
│    else:                                                        │
│       → ngx_close_connection()                                  │
│       → remove from epoll/kqueue                                │
│       → close(2) socket                                         │
│       → destroy connection from pool                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Performance Characteristics

### Concurrency Model

| Aspect | Details |
|--------|---------|
| **Workers** | Pre-forked (N = CPU cores by default) |
| **Per-Worker Concurrency** | Event-driven (10K-100K connections per worker typical) |
| **Context Switching** | O(1) per event via epoll; no threads |
| **Memory per Connection** | ~2-4 KB (connection struct + buffers) |
| **Scalability Limit** | File descriptor ulimit (typically 65536 per process) |

### Hot Paths

1. **Event Loop** (line 195 in `ngx_event.c`): Called per event iteration
   - Measure: `ngx_process_events_and_timers` should complete in < 100ms
   
2. **Accept (per new connection)**: `ngx_handle_accept_event()`
   - Measure: accept(2) + ngx_connection_t allocation
   
3. **HTTP Request Parsing** (line 108 in `ngx_http_parse.c`): Character-by-character
   - Measure: Parsing speed >> network speed (CPU rarely bottleneck)
   
4. **Buffer I/O** (writev/sendfile): Scatter-gather output
   - Measure: Zero-copy when possible (sendfile on file responses)

### Cache Behavior

**False Sharing Risk** ⚠️:
- Accept mutex (spinlock) on same cache line as other state
- Multiple workers spinning on mutex = cache line thrashing
- Mitigation: `NGX_CACHELINE_SIZE` alignment (64-128 bytes)

**L3 Cache Efficiency**:
- Connection pools fit in L3 (typical 8-20 MB per worker)
- HTTP parsing works on small buffers (4-8 KB, L1/L2 friendly)

---

## Known Limitations & Trade-Offs

### 1. **Accept Mutex Contention**
- **Problem**: Single mutex for all workers limits accept rate
- **Trade-off**: Avoids thundering herd but adds lock overhead
- **Mitigation**: `reuseport` (SO_REUSEPORT) allows kernel to distribute accepts
- **Validation**: Monitor `ngx_accept_mutex_held` time; should be < 1ms per cycle

### 2. **Single-Threaded Workers**
- **Problem**: One slow request blocks the worker
- **Trade-off**: Simplicity vs. parallelism within a worker
- **Mitigation**: Async operations (thread pools for file I/O, subrequests)
- **Validation**: If request handler runs > 100ms, investigate blocking calls

### 3. **Memory Pools Assume Bulk Cleanup**
- **Problem**: Leaks in pools are hard to diagnose
- **Trade-off**: O(1) deallocation vs. per-object lifetime
- **Mitigation**: `NGX_DEBUG` memory tracking; valgrind integration
- **Validation**: Monitor memory growth with `top` or `/proc/[pid]/status`

### 4. **No NUMA Awareness** (Future Work)
- **Problem**: Allocations may cross NUMA nodes on multi-socket systems
- **Trade-off**: Generic vs. platform-specific optimization
- **Mitigation**: Manual CPU affinity (worker to socket)
- **Validation**: Benchmark with `numactl` on NUMA systems

### 5. **HTTP/1.1 Keep-Alive Doesn't Scale**
- **Problem**: Each connection holds memory even when idle
- **Trade-off**: Reuse vs. connection limit
- **Mitigation**: `keepalive_timeout` (default 75s); HTTP/2 multiplexing
- **Validation**: Monitor active connections vs. expected peak

---

## Configuration Tuning Points

| Directive | Scope | Impact | Default |
|-----------|-------|--------|---------|
| `worker_processes` | Main | Number of workers | CPU cores |
| `worker_connections` | Events | Max connections per worker | 512 |
| `worker_rlimit_nofile` | Main | File descriptor limit | OS default |
| `keepalive_timeout` | HTTP | Keep-alive idle timeout | 75s |
| `client_header_buffer_size` | HTTP | Request header buffer | 1KB |
| `large_client_header_buffers` | HTTP | Oversized header buffer | 32KB (4 buffers) |
| `timer_resolution` | Main | Timer coarseness | 0 (high precision) |
| `multi_accept` | Events | Accept multiple per cycle | off |
| `reuseport` | Listen | Enable SO_REUSEPORT | off |

**Tuning Guide**:
- **High throughput**: Increase `worker_connections` (e.g., 10K)
- **Memory-constrained**: Reduce `client_header_buffer_size`; enable `reuseport` for better accept scalability
- **Latency-sensitive**: Decrease `timer_resolution` (add CPU overhead)

---

## Invariants & Correctness

### Critical Invariants

1. **Connection State Machine**:
   - Connections are always in a valid state (initialized, reading, writing, closing)
   - No access to closed connections (checked via `destroyed` flag)

2. **Event Loop Termination**:
   - Event loop must exit within reasonable time under graceful shutdown
   - Checked by `ngx_event_no_timers_left()` (line 713 in ngx_process_cycle.c)

3. **Memory Pool Consistency**:
   - Pool `current` pointer always valid and within block boundaries
   - Large allocations tracked independently (no coalescing)

4. **Signal Safety**:
   - All signal handlers set atomic flags only; no locks, malloc, or I/O
   - Main loops check flags between events

5. **Accept Mutex Fairness**:
   - Accept mutex is held < accept_mutex_delay (default 500ms)
   - If held longer, worker is slow; other workers skip accept

### Validation Methods

- **ASAN/Valgrind**: Memory corruption detection
- **Strace**: Trace system calls to verify I/O patterns
- **Perf**: Flame graphs to identify hot functions
- **Load testing**: Verify stability under sustained load

---

## Next Steps for Understanding

1. Read **MEMORY_AND_CONCURRENCY.md** for deep dive on worker pooling and shared state
2. Read **BUFFER_MANAGEMENT.md** for buffer allocation and overflow risks
3. Read **CODE_WALKTHROUGH_EVENT_LOOP.md** for annotated source walk-through
4. Review **ASSUMPTIONS_AND_THREATS.md** for threat model and validation methods
5. Study **PERFORMANCE_BOTTLENECKS.md** for optimization opportunities

---

**References**:
- nginx source: https://github.com/nginx/nginx (analyzed at src/core/nginx.c, src/event/ngx_event.c)
- Event system: epoll(7), kqueue(2), select(2) man pages
- POSIX signals: signal-safety(7), sigaction(2)
- Memory pools: "Scalable Memory Allocation Using jemalloc" (inspired pool design)

