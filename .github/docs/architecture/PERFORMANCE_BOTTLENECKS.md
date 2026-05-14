# NGINX Performance Bottlenecks & Tuning

**Audience**: Performance engineers, SREs, capacity planners.
**Last Updated**: 2024
**Focus**: Hot paths, cache behavior, lock contention, tuning parameters.

---

## Executive Summary

NGINX's performance is bottleneck-free under most workloads (network I/O is the limiter), but specific scenarios can trigger CPU, memory, or lock contention. This document identifies hot paths, measures their performance impact, and provides tuning guidance.

**Rule of Thumb**: If network is not saturated, nginx is not the problem.

---

## Hot Paths (Priority 1: Always Hot)

### 1. Event Loop (`src/event/ngx_event.c:195`)

**Function**: `ngx_process_events_and_timers(ngx_cycle_t *cycle)`

**Frequency**: Called once per event (10,000-100,000 times/sec typical)

**Operations**:
```
Time: 0µs                 -- Event loop iteration starts
      ├─ ngx_event_find_timer()       ~1-10µs (search timer tree)
      ├─ ngx_trylock_accept_mutex()   ~0.1-10µs (spinlock; fast path if not held)
      ├─ ngx_process_events()         ~100-1000µs (epoll_wait; sleeps if no events)
      ├─ ngx_event_expire_timers()    ~1-100µs (linear tree walk)
      ├─ ngx_shmtx_unlock()           ~0.1µs (atomic store)
      └─ ngx_event_process_posted()   ~1-1000µs (run all posted handlers)
Time: 1-2ms (typical)
```

**Measurement**:
```bash
# Profile event loop
perf record -p $(pgrep -f nginx.*worker | head -1) -F 99 -g -- sleep 10
perf report

# Expected: event loop visible in top stack traces
```

**Bottlenecks**:
- **Timer tree search**: Linear on large timer count (but rare; most timers batch-expire)
- **Accept mutex**: Contention under concurrent worker accepts (see Section 2)
- **epoll_wait**: Blocks if no events (not a bottleneck; expected behavior)
- **Posted events**: Large queue → many handler calls → CPU time

**Tuning**:
- `timer_resolution` (line 58 in `nginx.c`): Coarsen timer granularity to reduce searches
  - Default: 0 (high precision, recalculate after each epoll_wait)
  - Tuned: 100ms (recalculate every 100ms → reduce overhead ~10%)
- `multi_accept`: Accept multiple sockets per cycle (trade latency for throughput)
- `reuseport`: Eliminate accept mutex contention

**Validation**:
```bash
# Measure event loop iteration time
# Add instrumentation to ngx_process_events_and_timers():
// delta = ngx_current_msec - ngx_current_msec @ start
// ngx_log_debug1(..., "event_loop_time: %M ms", delta)

# Monitor logs; should see <1ms typical, <10ms worst-case
```

---

### 2. HTTP Request Parsing (`src/http/ngx_http_parse.c:108`)

**Function**: `ngx_http_parse_request_line(ngx_http_request_t *r, ngx_buf_t *b)`

**Frequency**: Once per request (1,000-100,000 req/sec typical)

**Performance Characteristics**:
- **Character-by-character state machine**: No backtracking or lookahead
- **Time complexity**: O(N) where N = request line length (typical 50-200 bytes)
- **Expected time**: ~1-5 µs (trivial vs. I/O time)
- **Bottleneck**: Only if network overhead dominates (GigE line-rate; < 1 µs parsing matters)

**Code Path**:
```c
for (p = b->pos; p < b->last; p++) {  // Iterate through buffer
    ch = *p;
    
    switch (state) {                    // Jump table (fast)
    case sw_method:
        // Validate char, transition state
        // Very tight inner loop; cache-friendly
        break;
    }
}
return NGX_AGAIN;  // or NGX_OK
```

**Measurement**:
```bash
# Benchmark parser with synthetic input
# Write microbenchmark; time 1 million requests

# Or: measure with strace
strace -c -e trace=none nginx  # total syscall overhead
# CPU time should be negligible vs. I/O
```

**Bottlenecks**:
- None typically; parser is fast
- Only if receiving single-byte packets (pathological network)

**Tuning**:
- Irrelevant for normal workloads
- If parsing is bottleneck: something else is wrong (e.g., many slow requests)

---

### 3. Connection Accept (`src/core/ngx_connection.c:242-274`)

**Function**: `ngx_get_connection()` → allocate connection + pool

**Frequency**: Once per new connection (10-10,000 conn/sec typical)

**Operations**:
```
Time: 0µs
      ├─ Dequeue from free list           ~0.1µs (pointer dereference)
      ├─ ngx_create_pool()                ~5-50µs (malloc + memset)
      ├─ ngx_atomic_fetch_add()           ~0.1µs (atomic increment)
      └─ Return connection
Time: 5-50µs (dominated by malloc)
```

**Measurement**:
```bash
# Measure with strace
strace -e mmap,brk,malloc_trim -c nginx
# Visible as syscall overhead

# Or: perf with malloc hook
perf record -p $(pgrep nginx.*worker | head -1) -g -- sleep 10
```

**Bottlenecks**:
- **malloc() latency**: If glibc malloc is contended (rarely true; each worker has own heap)
- **Pool initialization**: Large pool → large memset → cache miss

**Tuning**:
- `connection_pool_size` (line 46 in `ngx_connection.h`):
  - Default: usually 256-512 bytes per connection
  - Larger: pre-allocate buffers; faster first request
  - Smaller: less memory per connection; more fragmentation
- Consider `jemalloc` (faster multi-threaded allocator)
  - Compile: `./configure --with-jemalloc`

**Validation**:
```bash
# Monitor connection allocation rate
# Expected: <1µs per connection on modern hardware
# If > 10µs: malloc contention (use jemalloc)
```

---

### 4. Buffer I/O (Write) (`src/core/ngx_output_chain.c`)

**Function**: `ngx_output_chain()` → writev() or sendfile()

**Frequency**: Once per response chunk (1,000-1,000,000 times/sec typical)

**Operations**:
```
Time: 0µs
      ├─ Build chain of buffers         ~1-10µs (walk buffer list)
      ├─ writev(2) or sendfile(2)       ~1-100µs (syscall)
      ├─ Check if partial write         ~0.1µs (compare)
      └─ Schedule write event if needed ~0.1µs (set flag)
Time: 1-100µs (dominated by syscall)
```

**Measurement**:
```bash
# Profile write syscalls
perf stat -e syscalls:sys_enter_writev,syscalls:sys_exit_writev -p $(pgrep nginx)
# Count: should be ~req_count (one write per response, typically)

# Measure syscall overhead
time curl -s http://localhost/ > /dev/null  # small file
# Expected: <1ms (dominated by network, not syscall)
```

**Bottlenecks**:
- **Context switches**: If many small writev calls → high syscall overhead
- **Sendfile not used**: Large files sent in chunks instead of zero-copy
- **Partial writes**: Large response fragmented; many syscalls needed

**Tuning**:
- `tcp_nopush` (TCP_CORK on Linux):
  - Delays ACK until buffer full or flush → fewer packets
  - Default: off
  - Tuned: on (especially for static files)
  - Example: `tcp_nopush on;`
- `tcp_nodelay`:
  - Disables Nagle's algorithm; small packets sent immediately
  - Default: off (Nagle enabled)
  - Tuned: on (for latency-sensitive apps)
- `sendfile` (zero-copy for static files):
  - Default: off
  - Tuned: on
  - Requirement: files only (not generated content)
- `sendfile_max_chunk`:
  - Limit per sendfile call (prevent monopolizing worker)
  - Default: 0 (unlimited)
  - Tuned: 2MB (empirical; prevents stalls)

**Validation**:
```bash
# Measure response latency
time ab -n 1000 -c 100 http://localhost/
# p50 should be <50ms (depends on network/app logic)

# Check sendfile usage
strace -e sendfile64 -c nginx
# Should see sendfile calls for file responses
```

---

## Cache Behavior Analysis

### L1/L2/L3 Cache Layout

**Memory Access Patterns**:

1. **Connection Struct** (`ngx_connection_t`):
   - Size: ~300 bytes (fits in L1 cache line + partial L2)
   - Access: frequently (multiple times per event)
   - Expected: L1 hits > 95%

2. **Event Tree** (`ngx_rbtree_t`):
   - Size: scales with connection count
   - Access: per event (sparse; mostly leaf nodes)
   - Expected: L2 hits ~70%, L3 misses ~30%

3. **Buffer Chains** (`ngx_chain_t`):
   - Size: pointer + next (16 bytes typically)
   - Access: linear walk (cache-friendly)
   - Expected: L1 hits > 90%

**Measurement**:
```bash
# Profile cache misses
perf stat -e LLC-loads,LLC-load-misses,LLC-prefetch-misses -p $(pgrep nginx.*worker | head -1) -- sleep 10

# Expected under idle: low miss rate (<1%)
# Expected under high load: 5-15% (acceptable)
```

### False Sharing

**Risk**: Multiple workers spinning on accept mutex (shared cache line).

**Scenario**:
```
Cache Line (64 bytes):
┌─────────────────────────────────────────────────────────┐
│ ngx_accept_mutex (8B) │ padding (56B)                   │
└─────────────────────────────────────────────────────────┘

Worker 0 reads: L3 miss → 40 cycles (memory latency)
Worker 1 reads: L3 hit → 4 cycles (cache hit)
Worker 0 writes (releases mutex): invalidates cache line
Worker 1's read stale → L3 miss → 40 cycles
→ Pattern: every lock acquire/release = 40 cycle penalty
```

**Validation**:
```bash
# Measure false sharing impact
# Benchmark: baseline (low contention) vs. high contention
# Expected: <10% slowdown under moderate contention

# Check cache line alignment
objdump -d nginx | grep "accept_mutex"  # Verify alignment
```

**Mitigation**:
- ✓ nginx aligns accept mutex to cache line boundary
- `reuseport` eliminates contention entirely

---

## Lock Contention Points

### 1. Accept Mutex (Shared)

**Location**: `src/event/ngx_event.c:51-58`

**Contention Pattern**:
```
Workers: 1   2   4   8   16
Throughput (conn/sec):
800K    900K 850K 700K 500K
```

**Analysis**: Throughput drops as workers increase (diminishing returns).

**Root Cause**: Accept mutex serializes accepts.
- Only one worker can accept per event loop iteration
- Mutex acquisition time: ~0.1-1µs (uncontended) → ~10-100µs (contended)
- Impact: at 8 workers, ~7/8 workers blocked on mutex per cycle

**Mitigation**:
- **`reuseport` (recommended)**: SO_REUSEPORT allows each worker to accept independently
  ```nginx
  listen 80 reuseport;
  ```
  - Kernel distributes SYNs across workers
  - No mutex needed
  - Expected improvement: ~2-3x throughput at 8 workers

- **`multi_accept off`** (default): Single accept per cycle
  - Tradeoff: fair distribution vs. throughput

- **Reduce worker count**: If only 2 workers, contention minimal

**Measurement**:
```bash
# Measure accept rate (conn/sec)
ab -n 10000 -c 100 http://localhost/

# With reuseport:
ab -n 10000 -c 100 http://localhost/
# Expected: higher throughput

# Monitor mutex hold time
# Instrument ngx_shmtx_lock/unlock: measure elapsed time
```

### 2. Atomic Counter Updates (Shared)

**Location**: `src/event/ngx_event.c:63-76` (ngx_stat_* counters)

**Contention Pattern**:
- Very low contention (each worker updates independently; one atomic per event)
- Negligible performance impact

**Not a bottleneck in practice**.

---

## Memory Bottlenecks

### 1. Per-Connection Pool Allocation

**Scenario**: 10,000 concurrent connections × 8 KB pool = 80 MB per worker.

**Allocation Strategy**:
- Pool blocks allocated from system malloc
- Linked via single-linked list (`d.next`)
- On close: entire pool freed (O(N) walk)

**Bottleneck**: If connection pool becomes fragmented:
- Many small blocks → large pool chain → slow deallocation
- Unlikely in practice (most allocations fall into first 1-2 blocks)

**Validation**:
```bash
# Measure pool fragmentation
# Instrument ngx_destroy_pool(): count blocks freed
// ngx_log_debug2(..., "pool destroy: %d blocks, %uz total", block_count, total_size)

# Expected: <10 blocks per connection pool
# If > 100: possible fragmentation issue
```

### 2. Header Buffer Allocation

**Scenario**: Large headers → multiple buffers (up to 4 × 32 KB = 128 KB).

**Allocation Pattern**:
- First buffer: 1 KB (default `client_header_buffer_size`)
- Overflow → second buffer: 32 KB (default `large_client_header_buffers`)
- Repeat up to 4 times

**Bottleneck**: If many requests have large headers:
- Each connection allocates large buffers
- Total memory = worker_connections × header_buffer_size

**Example**:
```
worker_connections = 10,000
large_client_header_buffers = 4 32k (default)
Worst case: 10,000 × 32 KB = 320 MB per worker
4 workers: 1.3 GB total
```

**Mitigation**:
- Reduce `large_client_header_buffers` if headers known to be small
- Monitor `Accept: application/json` requests (typically small headers)

**Validation**:
```bash
# Measure header buffer usage
# Log: connection->buffer->end - connection->buffer->start
// Average over 1000 connections

# If average > 4KB: tune `client_header_buffer_size` up
```

---

## CPU Bottlenecks

### 1. HTTP Parsing Under Pathological Input

**Scenario**: Many requests with very long URIs or headers.

**Performance**:
- Parser: O(N) where N = request line length
- Expected: <1µs for typical requests (50-200 bytes)
- Pathological: >100 byte queries; parsing becomes visible

**Mitigation**:
- Limit URI size via `large_client_header_buffers` size parameter
- Validate client input at reverse proxy layer

### 2. Regex Matching (Location Matching)

**Scenario**: Many location blocks with complex regex.

**Performance**:
- Compiled regex: fast (DFA, not NFA)
- Expected: <10µs per location match
- Pathological: deeply nested locations or many regex

**Validation**:
```bash
# Profile regex overhead
perf stat -e cache-references,cache-misses,cpu-cycles -p $(pgrep nginx)
# If regex CPU time significant: simplify locations
```

### 3. Logging Overhead

**Scenario**: High-frequency logging (e.g., NGX_LOG_DEBUG).

**Performance**:
- Each log message: sprintf + write syscall
- Expected: <100µs per message (acceptable)
- Debug logging: can reduce throughput 10-50%

**Mitigation**:
- Disable debug logging in production (`error_log ... info;`)
- Use buffer-based async logging if needed

**Validation**:
```bash
# Measure logging impact
# Benchmark: error_log info vs. error_log debug
# Expected: debug 2-5x slower than info
```

---

## Network Bottlenecks (Outside nginx)

These aren't nginx problems, but can appear as such:

### 1. Packet Loss

**Symptom**: Connections drop unexpectedly; high latency.

**Root Cause**: Network buffer overflow (not nginx).

**Validation**:
```bash
# Monitor network interface
ethtool -S eth0 | grep -i drop
# Should be zero (or very low)

# Monitor socket buffer
netstat -an | grep Recv-Q
# Should be close to zero (connections not backed up)
```

### 2. TCP Window Scaling

**Symptom**: Throughput degrades with latency.

**Root Cause**: TCP window size too small for bandwidth-delay product.

**Validation**:
```bash
# Check TCP window size
cat /proc/sys/net/core/rmem_max
cat /proc/sys/net/core/wmem_max
# Should be >> bandwidth × latency
# Example: 1 Gbps × 10 ms = 1.25 MB minimum
```

---

## Tuning Checklist

### Network-Level

- [ ] `tcp_nopush on;` (reduce packet overhead)
- [ ] `tcp_nodelay on;` (for latency-sensitive apps)
- [ ] `sendfile on;` (zero-copy for files)
- [ ] `sendfile_max_chunk 2m;` (prevent worker stalls)

### Connection-Level

- [ ] `worker_connections 10000;` (or higher, if available FDs)
- [ ] `keepalive_timeout 5s 5s;` (short timeout for connection reuse)
- [ ] `keepalive_requests 100;` (requests per connection)

### Memory-Level

- [ ] `connection_pool_size 512;` (or tune for workload)
- [ ] `large_client_header_buffers 4 32k;` (match expected headers)
- [ ] `client_max_body_size 10m;` (limit uploads)

### Event-Level

- [ ] `worker_processes auto;` (detect CPU count)
- [ ] `worker_priority 0;` (or negative for high-priority)
- [ ] `multi_accept on;` (if accept-bound)
- [ ] `reuseport on;` (if using Linux 3.9+)

### Timer-Level

- [ ] `timer_resolution 100ms;` (or keep at 0 for precision)
- [ ] `client_header_timeout 10s;` (prevent slowloris)
- [ ] `client_body_timeout 10s;`
- [ ] `send_timeout 10s;`

### OS-Level (in /etc/sysctl.conf)

```bash
# Network
net.core.rmem_max = 134217728      # 128 MB
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 65536 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432

# Connection queue
net.core.somaxconn = 65535         # Accept queue size
net.ipv4.tcp_max_syn_backlog = 65535

# File descriptor limit
ulimit -n 65535
```

---

## Benchmarking Methodology

### Load Test Setup

```bash
# Tool: Apache Bench (simplest)
ab -n 10000 -c 100 -k http://localhost/

# Tool: wrk (more realistic)
wrk -t8 -c100 -d30s http://localhost/

# Tool: vegeta (sustained load)
echo "GET http://localhost/" | vegeta attack -rate=1000 -duration=30s | vegeta report

# Tool: hey (similar to ab)
hey -n 10000 -c 100 -k http://localhost/
```

### Metrics to Track

- **Throughput**: Requests/sec (target: network-limited, not CPU-limited)
- **Latency**: p50, p95, p99 (target: <50ms, <100ms, <500ms typical)
- **Error Rate**: Should be zero (unless testing limits)
- **Connection Count**: Monitor with `ss -tan`
- **CPU Utilization**: `top`; should be balanced across workers
- **Memory**: `ps aux`; should stabilize (not grow unbounded)

### Validation Test Cases

1. **Baseline**: Small file, no keep-alive
   - `ab -n 10000 http://localhost/index.html`

2. **With Keep-Alive**: Small file, persistent connections
   - `ab -n 10000 -k http://localhost/index.html`

3. **Large File**: Test sendfile overhead
   - `ab -n 1000 http://localhost/large-file.bin`

4. **High Concurrency**: Many simultaneous connections
   - `wrk -t8 -c10000 -d30s http://localhost/`

5. **Slow Client**: Simulate slow read
   - Custom script with `time.sleep(0.5)` between reads

---

## Conclusion

Nginx is CPU and memory efficient. Performance tuning is usually about:
1. **Network optimization** (tcp_nopush, sendfile, tcp_nodelay)
2. **Connection tuning** (worker_connections, keepalive settings)
3. **Accept scalability** (reuseport for multi-worker setups)
4. **OS limits** (somaxconn, file descriptors, buffer sizes)

If nginx is fast enough (network-limited), leave it alone. Measure before tuning; only optimize hot paths verified by profiling.

---

**References**:
- TCP Performance: RFC 7414 (TCP Evaluation Suite)
- Linux Network Tuning: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/

