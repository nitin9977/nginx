# NGINX Memory and Concurrency Model

**Audience**: Engineers designing high-concurrency systems, performance optimizers.
**Last Updated**: 2024
**Focus**: Worker isolation, shared state, allocation patterns, cache coherency.

---

## Executive Summary

NGINX uses a **process-based isolation model** where each worker is an independent OS process with its own memory space. This design eliminates thread-safety concerns but introduces inter-process communication overhead (accept mutex, signals). Memory is allocated from pools, enabling predictable deallocation and resistance to fragmentation.

**Key Properties**:
- **No data races** within a worker (single-threaded event loop)
- **Minimal shared state** (only accept mutex and atomic counters)
- **Zero intra-worker locking** (simplicity ≠ performance)
- **Predictable memory footprint** (pools vs. malloc)

---

## Process Model

### Worker Spawning

**File**: `src/os/unix/ngx_process_cycle.c:336` (`ngx_start_worker_processes`)

```c
static void
ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)
{
    // n = number of workers (from config: worker_processes)
    // type = NGX_PROCESS_WORKER (normal) or NGX_PROCESS_SINGLE (single mode, no fork)
    
    for (i = 0; i < n; i++) {
        ngx_spawn_process(cycle, ngx_worker_process_cycle,
                          (void *) (intptr_t) i, title, type);
        // Each call: fork() + exec callback in child
    }
}
```

**Fork Chain** (line 344 in `ngx_process_cycle.c`):
```c
ngx_spawn_process()
  ├─ fork(2)
  ├─ child:
  │   ├─ signal handlers reinstalled (only safe subset)
  │   ├─ close inherited pipes/sockets from master
  │   ├─ call callback: ngx_worker_process_cycle(cycle, worker_id)
  │   └─ run event loop forever (or until signal)
  │
  └─ parent:
       ├─ track child PID
       └─ continue master loop (wait for signals, reap children)
```

**Process Layout**:
```
Parent Process (Master)
  │
  │  [Shared Memory]
  │  ├─ accept_mutex (ngx_shmtx_t)
  │  ├─ atomic counters (ngx_stat_*)
  │  └─ configured by ngx_cycle_init_control_process_cycle() or master_process
  │
  ├─ Child 0 (Worker)     [Private Heap]
  │   └─ event loop, connections, buffers, pools
  │
  ├─ Child 1 (Worker)     [Private Heap]
  │   └─ event loop, connections, buffers, pools
  │
  └─ Child N (Worker)     [Private Heap]
       └─ event loop, connections, buffers, pools
```

### Per-Worker Memory Layout

**File**: `src/core/ngx_cycle.c`

Each worker inherits the `ngx_cycle_t` struct from master but operates independently:

```c
struct ngx_cycle_s {
    ngx_connection_t   *connections;   // array of ngx_connection_t
                                       // size = worker_connections
    ngx_event_t        *read_events;   // array of read events
    ngx_event_t        *write_events;  // array of write events
    
    ngx_pool_t         *pool;          // master pool (config, modules)
    ngx_list_t          listening_sockets;  // array of ngx_listening_t
    
    ngx_log_t          *log;           // logger
    // ... module-specific data in conf_ctx
};
```

**Size Calculation**:
```c
// Rough per-worker memory:
connections[]     = worker_connections * sizeof(ngx_connection_t)
                  = 10,000 * ~300 bytes = 3 MB
read_events[]     = worker_connections * sizeof(ngx_event_t)
                  = 10,000 * ~96 bytes = 960 KB
write_events[]    = 960 KB
total fixed       ≈ 5-6 MB per worker

// Plus: per-connection pools
connection_pools  = worker_connections * avg_pool_size
                  = 10,000 * 8 KB = 80 MB per worker
                  (highly variable; depends on request size)
```

**Invariant**: Each worker's memory is **independent**. No synchronization needed within a worker.

---

## Shared State & Synchronization

### 1. Accept Mutex (Shared Memory)

**File**: `src/core/ngx_shmtx.h`, `src/core/ngx_shmtx.c`

**Purpose**: Serialize accept() to avoid thundering herd (all workers waking up for one connection).

**Structure**:
```c
typedef struct {
    ngx_atomic_t  lock;               // spinlock (atomic compare-and-swap)
#if (NGX_HAVE_POSIX_SEM)
    ngx_atomic_t  wait;               // semaphore counter
    sem_t        *sem;                // POSIX semaphore (in shared memory)
#endif
} ngx_shmtx_sh_t;

typedef struct {
    ngx_shmtx_sh_t *sh;               // pointer to shared structure
    ngx_pid_t       pid;               // owning PID
#if (NGX_HAVE_POSIX_SEM)
    sem_t          *sem;               // semaphore in shared memory
#else
    ngx_uint_t      spin;              // spinlock count before sleep
#endif
} ngx_shmtx_t;
```

**Spinlock Behavior** (line 13 in `ngx_spinlock.c`):
```c
void
ngx_spinlock(ngx_atomic_t *lock, ngx_atomic_int_t value, ngx_uint_t spin)
{
    ngx_uint_t  i, n;
    
    for ( ;; ) {
        if (ngx_atomic_cmp_set(lock, 0, value)) {
            return;                    // lock acquired
        }
        
        if (ngx_cpu_pause_instruction_available()) {
            for (n = 1; n < spin; n <<= 1) {
                for (i = 0; i < n; i++) {
                    ngx_cpu_pause();   // PAUSE instruction (x86)
                }
                
                if (ngx_atomic_cmp_set(lock, 0, value)) {
                    return;
                }
            }
        }
        
        ngx_sched_yield();             // yield to other threads
    }
}
```

**Lock/Unlock** (line 13 in `ngx_shmtx.c`):

```c
ngx_int_t
ngx_shmtx_trylock(ngx_shmtx_t *mtx)
{
    return (ngx_atomic_cmp_set(mtx->sh->lock, 0, ngx_pid));
    // CAS: if lock==0, set lock=my_pid; return success
}

void
ngx_shmtx_lock(ngx_shmtx_t *mtx)
{
    ngx_spinlock(&mtx->sh->lock, (ngx_atomic_int_t) ngx_pid, mtx->spin);
    // Spin until CAS succeeds
}

void
ngx_shmtx_unlock(ngx_shmtx_t *mtx)
{
    if (mtx->sh->wait) {
        ngx_atomic_cmp_set(&mtx->sh->lock, (ngx_atomic_int_t) ngx_pid, 0);
        // Atomic clear
        ngx_sem_post(mtx->sem);        // wake one waiting worker
    } else {
        ngx_atomic_cmp_set(&mtx->sh->lock, (ngx_atomic_int_t) ngx_pid, 0);
    }
}
```

**Event Loop Usage** (`ngx_event.c:219-239`):

```c
if (ngx_use_accept_mutex) {
    if (ngx_accept_disabled > 0) {
        ngx_accept_disabled--;         // backpressure: decrement counter
    } else {
        if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
            return;                     // lock failed; skip this cycle
        }
        
        if (ngx_accept_mutex_held) {
            flags |= NGX_POST_EVENTS;   // got lock; defer event processing
        } else {
            // Lost lock race (rare); shorten timer to retry
            if (timer == NGX_TIMER_INFINITE || timer > ngx_accept_mutex_delay) {
                timer = ngx_accept_mutex_delay;  // default 500ms
            }
        }
    }
}
```

**Backpressure Mechanism** (line 53-57 in `ngx_event_accept.c`):

```c
ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;
// If (total_connections - free_connections) > 7/8 * total:
//   backpressure: skip accept for N cycles
//
// Purpose: Stop accepting when we're running out of connection slots
```

**Invariant**: Accept mutex is always released before returning to event loop (line 257-259 in `ngx_event.c`).

**Performance Impact** ⚠️:
- Mutex contention scales linearly with worker count
- Typical hold time: < 100 µs (single accept + post event)
- Mitigation: `SO_REUSEPORT` (kernel-distributed accept, no mutex needed)

### 2. Atomic Counters (Optional Statistics)

**File**: `src/event/ngx_event.c:61-77`

```c
#if (NGX_STAT_STUB)

static ngx_atomic_t   ngx_stat_accepted0;
ngx_atomic_t         *ngx_stat_accepted = &ngx_stat_accepted0;

static ngx_atomic_t   ngx_stat_handled0;
ngx_atomic_t         *ngx_stat_handled = &ngx_stat_handled0;

// ... more counters (requests, active, reading, writing, waiting)

#endif
```

**Used by**: Monitoring endpoints (ngx_http_stub_status_module), internal diagnostics.

**Access Pattern**: Atomic increment without locks.

```c
ngx_atomic_fetch_add(ngx_stat_accepted, 1);  // Worker 0: increment
ngx_atomic_fetch_add(ngx_stat_accepted, 1);  // Worker 1: increment (concurrent)
// Result: both increments visible, no data race
```

**Not Guarded**: These are "best effort" and can be slightly inaccurate under contention (acceptable for diagnostics).

### 3. Connection Counter

**File**: `src/event/ngx_event.c:47-48`

```c
static ngx_atomic_t   connection_counter = 1;
ngx_atomic_t         *ngx_connection_counter = &connection_counter;

// Allocated once, shared by all workers
```

**Usage** (line 242 in `ngx_connection.c`):

```c
c->number = ngx_atomic_fetch_add(ngx_connection_counter, 1);
// Unique connection ID across all workers (used for logging, debugging)
```

---

## Memory Allocation Strategy

### Pool-Based Allocation

**File**: `src/core/ngx_palloc.c`

**Philosophy**: Allocate and deallocate in bulk; avoid per-object fragmentation.

#### Pool Creation

```c
ngx_pool_t *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;
    
    // Allocate single aligned block
    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    // NGX_POOL_ALIGNMENT = 16 bytes (typical)
    
    // Initialize pool header (within the block)
    p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    p->d.end = (u_char *) p + size;
    p->d.next = NULL;
    p->d.failed = 0;
    
    // Threshold for "large" allocations (separate tracking)
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    
    // Other tracking
    p->current = p;                    // current block pointer
    p->large = NULL;                   // large allocations list
    p->cleanup = NULL;                 // cleanup handlers
    p->log = log;
    
    return p;
}
```

**Structure**:
```
Pool Block (e.g., 4 KB):
┌────────────────────────┐
│ ngx_pool_t header      │ ← sizeof(ngx_pool_t) ≈ 80 bytes
├────────────────────────┤
│                        │
│  Free Space            │ ← allocate from here
│  (p->d.last .. p->d.end)
│                        │
└────────────────────────┘
```

#### Allocation Path

```c
void *
ngx_palloc(ngx_pool_t *pool, size_t size)
{
    if (size <= pool->max) {
        return ngx_palloc_small(pool, size, 1);  // aligned small alloc
    }
    return ngx_palloc_large(pool, size);         // separate large block
}
```

**Small Allocation** (line 149 in `ngx_palloc.c`):

```c
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
{
    u_char      *m;
    ngx_pool_t  *p;
    
    p = pool->current;
    
    for ( ;; ) {
        m = p->d.last;
        
        if (align) {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);  // align to 8 or 16 bytes
        }
        
        if ((size_t) (p->d.end - m) >= size) {
            p->d.last = m + size;
            return m;                             // ✓ Allocated from current block
        }
        
        if (p->d.next == NULL) {
            return ngx_palloc_block(pool, size);  // allocate new block
        }
        
        p = p->d.next;
        p->d.failed++;                            // track failed attempts
    }
}
```

**Key Insight**: Linear scan through blocks; allocates from first block with enough space.

**Large Allocation** (line 214 in `ngx_palloc.c`):

```c
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;
    
    // Allocate from system (malloc or mmap)
    p = ngx_alloc(size, pool->log);
    
    // Track in large allocations list (for selective deallocation)
    large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    large->alloc = p;
    large->next = pool->large;
    pool->large = large;                         // prepend to list
    
    return p;
}
```

**Layout** (conceptual):

```
Pool Blocks (chained):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ header      │     │ header      │     │ header      │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ data (100B) │     │ data (200B) │     │ data (300B) │
├─────────────┤     ├─────────────┤     ├─────────────┤
│ free (3900B)│     │ free (3800B)│     │ free (3700B)│
└─────────────┘     └─────────────┘     └─────────────┘
      ↑                   ↑
      └─ linked via d.next

Large Allocations (list):
┌─────────────────────┐
│ large pointer ──────→ [12 KB heap block]
│ large→next ────┐
│                ↓
└─────────────────────┘
     [24 KB heap block]
```

#### Deallocation

```c
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;
    
    // 1. Run cleanup handlers (e.g., close files, free custom data)
    for (c = pool->cleanup; c; c = c->next) {
        if (c->handler) {
            c->handler(c->data);
        }
    }
    
    // 2. Free large allocations
    for (l = pool->large; l; l = l->next) {
        if (l->alloc) {
            ngx_free(l->alloc);
        }
    }
    
    // 3. Free all pool blocks
    for (p = pool, n = pool->d.next; /* */; p = n, n = n->d.next) {
        ngx_free(p);
        if (n == NULL) break;
    }
}
```

**Complexity**: O(N) where N = number of blocks/large allocations. Usually small (< 100).

**Invariant**: Destroying a pool deallocates **all** memory associated with it (no leaks if cleanup complete).

### Connection Pool Lifecycle

**File**: `src/core/ngx_connection.c:242-258`

```c
ngx_connection_t *
ngx_get_connection(ngx_socket_t s, ngx_log_t *log)
{
    ngx_connection_t  *c;
    
    // Grab from free list
    c = ngx_cycle->free_connections;
    
    if (c == NULL) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "connection limit reached");
        return NULL;                              // ✗ At max connections
    }
    
    // Move from free list to active
    ngx_cycle->free_connections = c->data;
    ngx_cycle->free_connection_n--;
    
    // Allocate pool for this connection
    c->pool = ngx_create_pool(ngx_cycle->pool_size, log);
    // ngx_cycle->pool_size typically 4-8 KB per connection
    
    c->number = ngx_atomic_fetch_add(ngx_connection_counter, 1);
    // Unique ID
    
    return c;
}
```

**Connection Reuse**:
```c
void
ngx_free_connection(ngx_connection_t *c)
{
    // Destroy pool (all buffers, parsed headers, etc. freed)
    ngx_destroy_pool(c->pool);
    c->pool = NULL;
    
    // Return to free list
    c->data = ngx_cycle->free_connections;
    ngx_cycle->free_connections = c;
    ngx_cycle->free_connection_n++;
}
```

**Invariant**: Each active connection has exactly one pool. Connection pool is **not** reused; only the connection struct itself is recycled.

---

## Cache Coherency & Performance

### False Sharing Risk

**File**: `src/core/ngx_shmtx.c` (accept mutex alignment)

**Problem**: Multiple workers spinning on the same cache line causes thrashing.

**Example**:
```
Cache Line (64 bytes):
┌──────────────────────────────────────────────────────┐
│ ngx_accept_mutex.lock (8B) │ other state (56B)       │
└──────────────────────────────────────────────────────┘

Worker 0 reads lock → L3 cache miss → pulls line from memory
Worker 1 reads lock → gets cached copy
Worker 1 writes to different field in same line → invalidates line
Worker 0's cached copy is stale → re-fetches → L3 cache miss
→ Repeat 1000x/sec
→ Memory bus congestion
```

**Mitigation** (`src/core/ngx_config.h`):
```c
#define NGX_CACHELINE_SIZE  64  // or 128 for some systems

// Accept mutex placed in own cache line
typedef struct {
    ngx_shmtx_t  mutex;
    u_char       padding[NGX_CACHELINE_SIZE - sizeof(ngx_shmtx_t)];
} ngx_accept_mutex_block_t;
```

**Validation** ⚠️:
- Measure via `perf stat -e LLC-loads,LLC-load-misses` while idle
- Expected: Low miss rate if no contention
- Problem: High misses despite low accept rate → false sharing

### Per-Connection Allocation Strategy

**Goal**: Keep hot data (connection struct, buffer headers) in L1/L2 cache.

**Connection Size**: ~300 bytes (fits in L1 cache line + partial L2).

**Buffer Allocation**: 
- Initial read buffer: 4 KB (typical)
- Header buffer: on-demand, part of connection pool
- Keep together to maximize cache utilization

---

## Worker Independence & Isolation

### No Shared Connection State

**Invariant**: Connection `c` is owned exclusively by worker `i`.

**Consequence**: No locks needed for per-connection state.

```c
// Worker 0
c->read_buffer->pos += n;        // ✓ No lock needed (only worker 0 accesses)

// Worker 1
c->write_buffer->last += m;      // ✓ Independent connection

// ANY Worker
ngx_accept_disabled++;            // ✗ Needs atomic (shared)
```

### Event Queue Independence

**File**: `src/event/ngx_event.h:22-28`

```c
extern ngx_queue_t  ngx_posted_accept_events;
extern ngx_queue_t  ngx_posted_events;

// Per-worker instances; not shared
// Each worker has its own queue, processed independently
```

### Signal-Based Communication

**Master → Worker**: Signals (SIGHUP, SIGTERM, etc.)
- Async-safe; sets atomic flags
- Worker checks flags in event loop

**Worker → Master**: Not directly possible
- Use channel (ngx_channel_t) for acks
- Master polls channels during reap phase

---

## Concurrency Patterns

### 1. Accept Serialization

```
Timeline:
Time    Worker 0              Worker 1
────────────────────────────────────────
0ms     trylock()? NO
        sleep(500ms)          
                              trylock()? YES
                              lock acquired
                              accept() → new connection
                              process_events()
                              unlock()
500ms   trylock()? YES
        lock acquired
        no accept event (none pending)
        unlock()
```

**Pattern**: Only one worker processes accepts per event loop cycle.

### 2. Connection Ownership Transfer

**Stream Module** (upgrades from HTTP):
- HTTP worker1 accepts connection
- Executes ngx_stream_init_connection() → ownership transferred to stream subsystem
- Stream worker processes connection

**Pattern**: No transfer; connections don't move between workers.

### 3. Cache Manager Process (Special)

```c
ngx_start_cache_manager_processes(cycle, respawn)
{
    // Spawn cache manager + cache loader
    // Single process (no accept mutex needed)
    // Periodic maintenance: delete expired cache entries, etc.
}
```

**Isolation**: Cache manager runs outside normal worker event loop.

---

## Memory Safety Analysis

### Potential Data Races

**Scenario 1: Atomic Counter Increment**
```c
// Multiple workers
ngx_atomic_fetch_add(ngx_stat_accepted, 1);
// ✓ Safe: atomic operation

// vs. naive approach:
ngx_stat_accepted++;  // ✗ RACE: non-atomic read-modify-write
```

**Scenario 2: Shared Mutex Lock**
```c
// Multiple workers
ngx_shmtx_lock(&ngx_accept_mutex);
// ✓ Safe: serialized by kernel

// vs. broken approach:
if (ngx_accept_mutex_flag) return;  // ✗ TOCTOU race
ngx_accept_mutex_flag = 1;
```

**Scenario 3: Connection Pool Access**
```c
// Worker 0 only
c->pool->data ...      // ✓ Safe: no one else accesses c
```

### Use-After-Free Prevention

**Design**: Connection structs reused; pools destroyed on close.

```c
c = get_connection();           // ✓ Allocation
c->pool = ngx_create_pool();    // ✓ Pool created
... process request ...
free_connection(c);             // ✓ Free pool
// c is now on free list; can be reallocated
c2 = get_connection();          // c2 may == c
// c2->pool is new; old pool data gone
```

**Validation**: Valgrind with `--read-from-freed=yes` to catch UAF.

### Buffer Overflow Risk

**Allocation Bounds Check**:
```c
ngx_buf_t *b = ngx_create_temp_buf(pool, 4096);
// b->start, b->end correctly set
// b->pos = b->start, b->last = b->start

// User writes:
ngx_memcpy(b->pos, data, len);
// ✗ If len > 4096: overflow into next pool block

// Safe approach:
if (b->last - b->pos < len) {
    return NGX_AGAIN;  // need more space
}
```

**Validation**: ASAN (AddressSanitizer) catches buffer overflows.

---

## Configuration Impact on Concurrency

| Directive | Per-Worker Memory | Contention | Throughput |
|-----------|-------------------|-----------|-----------|
| worker_processes 1 | N/A | None | Limited (1 core) |
| worker_processes 8 | 5× (8 workers) | Accept mutex | High |
| worker_connections 1000 | 3 MB | Event loop | Good |
| worker_connections 100000 | 300 MB | Backpressure | Excellent (if enough RAM) |
| keepalive_timeout 75s | Steady | Lower (idle connections) | Depends on client count |
| keepalive_timeout 5s | Churns fast | Higher accept rate | Depends on application |

---

## Debugging Concurrency Issues

### Tools

1. **strace**: Trace system calls per process
   ```bash
   strace -p <worker_pid> -f
   ```
   Look for: accept(2), epoll_wait(2), write(2) patterns

2. **perf**: Profiling and tracing
   ```bash
   perf record -p <worker_pid> -e context-switches,cpu-migrations
   perf report
   ```
   Detect: excessive context switches (lock contention)

3. **Valgrind**: Memory error detection
   ```bash
   valgrind --leak-check=full nginx
   ```

4. **Custom logs**: Add debug output
   ```c
   ngx_log_debug4(NGX_LOG_DEBUG_EVENT, c->log, 0,
                  "accept_disabled: %d, mutex held: %d, free: %d, total: %d",
                  ngx_accept_disabled, ngx_accept_mutex_held,
                  ngx_cycle->free_connection_n, ngx_cycle->connection_n);
   ```

### Common Issues

**Issue**: High CPU, low throughput
- **Cause**: Spinlock contention on accept mutex
- **Check**: `perf stat` for L3 cache misses
- **Fix**: Enable `reuseport`; reduce worker count

**Issue**: Memory grows unbounded
- **Cause**: Pool leak (cleanup handler not registered)
- **Check**: `valgrind --leak-check=full`, monitor `/proc/[pid]/status`
- **Fix**: Register cleanup handlers with `ngx_pool_cleanup_add()`

**Issue**: Occasional crashes
- **Cause**: Data race (rare, usually signal-handling bug)
- **Check**: Run under ASAN; check signal handlers are safe
- **Fix**: Review `ngx_signal_handler()` for non-async-safe calls

---

## Next Steps

1. Review **BUFFER_MANAGEMENT.md** for per-connection buffer strategy
2. Study **CODE_WALKTHROUGH_EVENT_LOOP.md** for execution traces
3. Read **ASSUMPTIONS_AND_THREATS.md** for race condition analysis

---

**References**:
- Memory pools: Scott Meyers' "Effective C++"
- Concurrency: "Concurrent Programming in Java" (principles apply)
- POSIX atomics: GCC builtin atomic operations

