# NGINX Core Module Ramp-Up Guide

**Audience**: Engineers onboarding to `src/core` internals  
**Scope**: Architecture, module integration, and code walkthrough for representative core flows  
**Date**: 2026-05-19  
**Source paths**: All `src/` references are relative to the **nginx repository root** (`<repo>/src/`), not relative to `.github/`.

---

## 1. Architecture and Design of `src/core`

### 1.1 What `src/core` owns

The `src/core` subsystem is nginx's process-runtime foundation. It provides:

- Process startup and command-line handling (`nginx.c`)
- Global cycle lifecycle and reconfiguration (`ngx_cycle.c`)
- Module registration and configuration framework plumbing (`ngx_module.c`, `ngx_conf_file.c`)
- Connection object lifecycle and fd bookkeeping (`ngx_connection.c`)
- Memory allocators: pool allocator (`ngx_palloc.c`), slab (`ngx_slab.c`), shared-memory mutexes (`ngx_shmtx.c`)
- Core containers: array, list, queue, rbtree, radix tree, hash
- Logging (`ngx_log.c`), time cache (`ngx_times.c`), file helpers (`ngx_file.c`), resolver (`ngx_resolver.c`), parsing primitives (`ngx_parse.c`, `ngx_string.c`)

A practical way to read core is to split it into four planes:

1. **Boot plane**: `nginx.c` and `ngx_cycle.c`
2. **Runtime plane**: connection / log / time / pool subsystems
3. **Config plane**: conf parser + module `create_conf` / `init_conf` / `merge` hooks
4. **Utility plane**: containers, parsing, hashing, locks

### 1.2 Core design objects

#### `ngx_cycle_t` (`src/core/ngx_cycle.h`)

The global runtime snapshot. Key fields:

- `conf_ctx`: four-level pointer tree (`void ****`) — `conf_ctx[module->index]` holds module config
- `pool`: cycle-lifetime memory pool
- `modules` / `modules_n`: ordered module array built at init
- `connections` / `read_events` / `write_events`: pre-allocated parallel arrays, sized by `connection_n`
- `free_connections` / `free_connection_n`: intrusive free-list head and count
- `listening`: array of `ngx_listening_t` representing bound sockets
- `shared_memory`: list of `ngx_shm_zone_t` zones
- `old_cycle`: back-pointer used during reconfiguration to migrate state

#### `ngx_connection_t` (`src/core/ngx_connection.h`)

Transport endpoint state per socket:

- `data`: opaque pointer to protocol-specific session (HTTP request, mail session, etc.)
- `read` / `write`: pointers into per-worker `ngx_event_t` arrays
- `fd`: socket descriptor
- `recv` / `send` / `recv_chain` / `send_chain`: I/O function pointers (platform-specific, set at accept)
- `listening`: back-pointer to `ngx_listening_t`
- `pool`: per-connection memory pool
- `number`: monotonic connection counter (atomic)
- `queue`: reusable-connection queue node
- `idle`, `reusable`, `close`: state flags for keepalive lifecycle

#### `ngx_pool_t` (`src/core/ngx_palloc.h`)

Region allocator. Key fields:

- `d.last` / `d.end`: current block's bump-allocation window
- `d.next`: linked list of additional pool blocks
- `d.failed`: counter of allocation failures on this block (triggers `current` advance)
- `max`: threshold for small-vs-large allocation path
- `current`: pointer to first block still worth trying for small allocations
- `large`: linked list of individually `malloc`'d large allocations
- `cleanup`: linked list of cleanup callbacks run on `ngx_destroy_pool()`

#### `ngx_command_t` (`src/core/ngx_conf_file.h`)

Configuration directive descriptor:

- `name`: directive string
- `type`: bitmask encoding argument count + valid context (`NGX_MAIN_CONF`, `NGX_HTTP_SRV_CONF`, etc.)
- `set`: handler function invoked during parse
- `conf` / `offset`: struct offset mechanics for `ngx_conf_set_*` helpers

#### `ngx_listening_t` (`src/core/ngx_connection.h`)

Bound socket metadata:

- `fd`: listening socket descriptor
- `handler`: callback for accepted connections (e.g. `ngx_http_init_connection`)
- `servers`: opaque pointer to protocol-specific address array
- `connection`: the `ngx_connection_t` bound to this listening socket
- `reuseport`, `deferred_accept`, `backlog`, `pool_size`: tuning knobs

### 1.3 Bootstrap architecture

`main()` in `src/core/nginx.c` (line 197) performs the critical bootstrap sequence:

1. `ngx_get_options()` — parse CLI arguments
2. `ngx_time_init()` — initialize cached time
3. `ngx_log_init()` — open initial error log
4. `ngx_ssl_init()` — OpenSSL global init (if compiled)
5. Build temporary `init_cycle` with pool and log
6. `ngx_os_init()` — platform detection (pagesize, ncpu, rlimits)
7. `ngx_crc32_table_init()` — requires cacheline size from OS init
8. `ngx_add_inherited_sockets()` — hot-upgrade socket inheritance
9. `ngx_preinit_modules()` — assign module indices
10. `ngx_init_cycle(&init_cycle)` — full config parse and runtime setup
11. `ngx_init_signals()` — install signal handlers
12. `ngx_daemon()` — daemonize if configured
13. `ngx_create_pidfile()` — write PID file
14. Enter `ngx_single_process_cycle()` or `ngx_master_process_cycle()`

This means most runtime invariants are established before worker loops start.

### 1.4 Reconfiguration model

`ngx_init_cycle()` in `src/core/ngx_cycle.c` (line 39) constructs a new cycle from old cycle state:

1. Create new pool and cycle object
2. Copy prefixes, paths, and configuration file references
3. Initialize open files, shared memory, and listening arrays
4. Call `create_conf` on every `NGX_CORE_MODULE`
5. Parse config file via `ngx_conf_parse()`
6. Call `init_conf` on every core module
7. Open log files, create shared memory zones, open listening sockets
8. Invoke `init_module` callbacks on all modules

Important property: config reload is modeled as create-new-cycle then swap, not in-place mutation. Old cycle memory is reclaimed after old connections drain.

### 1.5 Connection lifecycle model

Connection pool management in `src/core/ngx_connection.c` is free-list based:

#### `ngx_get_connection()` (line 1207)

1. Call `ngx_drain_connections()` to reclaim reusable connections if free list is empty
2. Pop head from `cycle->free_connections` linked list
3. Zero the `ngx_connection_t` struct but preserve `read`/`write` event pointers
4. Zero both event structs, toggle `rev->instance` bit (stale-event guard)
5. Set `rev->data = c`, `wev->data = c`, `wev->write = 1`

#### `ngx_close_connection()` (line 1286)

1. Delete read/write timers if set
2. Remove events from kernel: `ngx_del_conn()` or individual `ngx_del_event()` calls
3. Remove from posted event queues if posted
4. Mark both events `closed = 1`
5. Remove from reusable connection queue
6. Call `ngx_free_connection()` to return slot to free list
7. Close socket fd

### 1.6 Pool allocator internals

`ngx_palloc()` in `src/core/ngx_palloc.c`:

- **Small path** (`size <= pool->max`): `ngx_palloc_small()` walks the `current` block chain, bumps `d.last`, returns pointer. If all blocks are exhausted, calls `ngx_palloc_block()` to allocate a new block and chain it.
- **Large path** (`size > pool->max`): `ngx_palloc_large()` calls `ngx_alloc()` (raw `malloc`) and links result into `pool->large` list.
- **Destroy**: `ngx_destroy_pool()` runs cleanup handlers, frees all large allocations, then frees all blocks in chain order.
- **Lazy advance**: when `ngx_palloc_small()` fails on a block, it increments `d.failed`. Once `failed > 4`, `pool->current` advances past that block permanently.

---

## 2. Integration and Dependencies

### 2.1 Integration with `src/os`

Core calls `ngx_os_init()` early and depends on OS-provided values for:

- `ngx_pagesize` — used by pool and slab allocators
- `ngx_cacheline_size` — used by shared-memory layout and CRC table init
- `ngx_ncpu` — used for worker process defaults
- `ngx_max_sockets` — RLIMIT_NOFILE value for fd budgeting

Many core tables and allocators assume these are initialized first.

### 2.2 Integration with `src/event`

Core owns connection object allocation, while event owns readiness orchestration.

Key seam:

- Event accept path acquires connections through `ngx_get_connection()`
- Event process init allocates `cycle->connections`, `cycle->read_events`, `cycle->write_events` arrays
- Connection close path removes event/timer registrations via `ngx_del_event` or `ngx_del_conn`
- `instance` bit toggling in `ngx_get_connection()` is critical for stale readiness suppression in epoll/kqueue

This coupling is strict: event handlers assume core lifecycle invariants around connection slots.

### 2.3 Integration with protocol modules (`http`, `mail`, `stream`)

Core config framework exposes generic module lifecycle; protocol modules plug in through `ngx_module_t` definitions and context callbacks.

Main core dependencies consumed by higher layers:

- `ngx_pool_t` — all per-request/session memory
- `ngx_log_t` — error reporting and debug logging
- `ngx_conf_t` / `ngx_command_t` — directive parsing and registration
- `ngx_connection_t` / `ngx_listening_t` — transport endpoint state
- Containers: `ngx_array_t`, `ngx_list_t`, `ngx_queue_t`, `ngx_rbtree_t`, `ngx_hash_t`

### 2.4 Integration with shared memory and locking

Core provides `ngx_shmtx_t` / `ngx_spinlock()` / `ngx_slab_pool_t` primitives used by event and protocol modules for cross-process coordination.

Concurrency assumption: workers are separate processes, not shared-address-space threads; shared memory + atomics are explicit opt-in paths. There are no in-process mutexes protecting connection or pool state.

---

## 3. Basic Code Walkthrough for Core Flows

### 3.1 Generic tracing template

Use this template when tracing any core behavior:

1. Identify the owning cycle context (old/new cycle, worker or master).
2. Check allocation domain (temporary pool vs cycle pool vs shared memory).
3. Trace module callback entrypoint (`create_conf` / `init_conf` / `init_module` / `init_process`).
4. Confirm cleanup path under both success and error branches.

### 3.2 Concrete walkthrough: startup to process-cycle handoff

#### Step A: `main()` builds init context

- `ngx_get_options()` parses `-c`, `-s`, `-p`, `-g` flags
- `ngx_time_init()` caches current time into slot array
- `ngx_log_init()` opens stderr or configured error log
- `init_cycle.pool = ngx_create_pool(1024, log)` — temporary pool for bootstrap
- `ngx_os_init(log)` — fills `ngx_pagesize`, `ngx_cacheline_size`, `ngx_ncpu`

#### Step B: `ngx_init_cycle()` builds runtime cycle

1. Allocate new pool (`NGX_CYCLE_POOL_SIZE`) and `ngx_cycle_t`
2. Copy `conf_prefix`, `prefix`, `error_log`, `conf_file` from old cycle
3. Initialize `paths`, `config_dump`, `open_files`, `shared_memory`, `listening` arrays
4. Build conf context: `cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *))`
5. For each `NGX_CORE_MODULE`: call `module->create_conf(cycle)`
6. `ngx_conf_parse()` — tokenize and execute config directives
7. For each core module: call `init_conf`
8. Open listening sockets via `ngx_open_listening_sockets()`
9. For each module: call `init_module`

#### Step C: process mode selection

- `ccf->master && ngx_process == NGX_PROCESS_SINGLE` → set `NGX_PROCESS_MASTER`
- Install signal handlers via `ngx_init_signals()`
- Daemonize if `ccf->daemon` is set
- Write PID file
- Enter `ngx_single_process_cycle(cycle)` or `ngx_master_process_cycle(cycle)`

#### Step D: worker runtime

- `ngx_worker_process_init()` sets CPU affinity, rlimits, user/group, calls each module's `init_process`
- Main loop: `for (;;) { ngx_process_events_and_timers(cycle); }`
- Signal flags (`ngx_terminate`, `ngx_quit`, `ngx_reopen`) checked per iteration

### 3.3 Concrete walkthrough: connection allocation and teardown

#### Allocation path (accept)

1. `ngx_event_accept()` calls `accept()`/`accept4()`, gets raw fd `s`
2. Calls `ngx_get_connection(s, ev->log)`
3. Inside: pops from free list, zeroes connection and events, toggles instance bit
4. Caller allocates `c->pool = ngx_create_pool(ls->pool_size, ...)`
5. Sets `c->recv = ngx_recv; c->send = ngx_send;` etc.
6. Calls `ls->handler(c)` — protocol takes ownership

#### Teardown path (protocol close)

1. Protocol calls `ngx_close_connection(c)` (or module-specific wrapper that also destroys pool)
2. Inside: timers removed, events deregistered, posted events removed
3. `ngx_free_connection(c)` pushes slot back onto free list
4. `close(fd)` — fd released to OS

### 3.4 Concrete walkthrough: pool allocation flow

1. `ngx_create_pool(4096, log)` — allocates 4096-byte aligned block, sets `d.last` after `ngx_pool_t` header, `max` to `min(block_remaining, ngx_pagesize - 1)`
2. `ngx_palloc(pool, 64)` — size ≤ max → `ngx_palloc_small()` → bumps `d.last` by 64 (aligned), returns pointer
3. `ngx_palloc(pool, 8192)` — size > max → `ngx_palloc_large()` → `malloc(8192)`, links into `pool->large`
4. Block exhaustion: when `d.last` can't fit request, `ngx_palloc_block()` allocates new block, chains via `d.next`, advances `pool->current` after 4+ failures
5. `ngx_destroy_pool(pool)` — runs all cleanup handlers, frees all large allocations, frees all blocks

---

## 4. Fast Mental Model for New Engineers

- **Cycle is the world**: all runtime state hangs off `ngx_cycle_t`
- **Pools are scoped lifetimes**: cycle pool outlives everything; connection pools die with the connection; request pools die with the request
- **Connections are pre-allocated slots**: sized once at startup, reused via free list
- **Modules plug in through callbacks**: `create_conf` → `init_conf` → `init_module` → `init_process`
- **Config reload = new cycle**: old cycle stays alive until old connections drain
- **No in-process locking**: workers are isolated processes; shared state uses shmem + atomics

---

## 5. Recommended First Read Order in `src/core`

1. `src/core/ngx_cycle.h` — understand `ngx_cycle_t` fields
2. `src/core/nginx.c` — `main()` bootstrap sequence
3. `src/core/ngx_cycle.c` — `ngx_init_cycle()` lifecycle
4. `src/core/ngx_palloc.h` and `src/core/ngx_palloc.c` — pool allocator
5. `src/core/ngx_connection.h` and `src/core/ngx_connection.c` — connection and listening lifecycle
6. `src/core/ngx_conf_file.h` and `src/core/ngx_conf_file.c` — config parser and directive framework
7. `src/core/ngx_module.h` — module type definitions and lifecycle callbacks
8. `src/core/ngx_buf.h` — buffer chain model (used everywhere)
9. `src/core/ngx_log.c` — logging internals

---

## 6. Practical Debug Checklist

When behavior seems wrong in a core flow, inspect in this order:

1. Is the pool lifetime correct? (cycle pool vs connection pool vs request pool)
2. Did `ngx_init_cycle()` complete successfully? Check for `NGX_CONF_ERROR` returns from module `init_conf` callbacks.
3. Is the connection free list exhausted? Check `cycle->free_connection_n`.
4. Did `ngx_close_connection()` fully clean up events, timers, and posted queue entries?
5. Was the `instance` bit toggled correctly on connection reuse? (stale epoll events)
6. Did a reload path reference memory from the old cycle after swap?
7. Are listening sockets opened and bound? Check `ngx_open_listening_sockets()` return.
8. Did an OS prerequisite (rlimit/pagesize/cacheline) fail during `ngx_os_init()`?

This checklist catches most real-world startup failures and connection lifecycle regressions.
