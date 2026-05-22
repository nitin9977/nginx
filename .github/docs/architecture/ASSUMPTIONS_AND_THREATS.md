# NGINX Assumptions & Threat Model

**Audience**: Architects, security engineers, deployment specialists.
**Last Updated**: 2024
**Purpose**: Enumerate non-obvious assumptions and associated threat scenarios.
**Source paths**: All `src/` references are relative to the **nginx repository root** (`<repo>/src/`), not relative to `.github/`.

---

## Executive Summary

This document catalogs 15 critical assumptions embedded in nginx's design, then for each: articulates failure scenarios, assesses impact, and prescribes validation methods. The goal is to move from conventional wisdom ("nginx is fast and stable") to **verified expectations** ("we've validated these assumptions under our workload").

---

## Assumption 1: Event Multiplexor is Available and Correct

**Assumption**: `epoll(2)`, `kqueue(2)`, or `select(2)` exists, correctly reports file descriptor readiness, and never loses events.

**Where Relied On**: 
- `src/event/modules/ngx_epoll_module.c:323` (epoll init)
- `src/event/ngx_event.c:248` (ngx_process_events)

**Failure Scenario**:
- epoll kernel bug → socket becomes readable but epoll doesn't report it
- Event starved → connection hangs indefinitely
- Slow client → worker stuck waiting for data that never arrives

**Impact**: **HIGH** – Worker becomes unavailable; connection hangs.

**Validation Method**:
```bash
# 1. Kernel version check
uname -r
# Known bad: Linux 2.6.32-279 (RHEL 6 early; epoll bug fixed in .279.47)

# 2. Stress test with partial reads
# Send HTTP request in small chunks (1 byte/sec) and verify timeout
python3 << 'EOF'
import socket, time, sys
s = socket.socket()
s.connect(('localhost', 80))
for ch in b'GET / HTTP/1.1\r\nHost: localhost\r\n\r\n':
    s.send(bytes([ch]))
    time.sleep(0.1)
    sys.stdout.write(f"sent {chr(ch)}\n")
time.sleep(5)
try:
    resp = s.recv(4096)
    print(f"Response: {resp[:100]}")
except socket.timeout:
    print("TIMEOUT (event loss suspected)")
EOF

# 3. Monitor with strace
strace -p $(pgrep -f "nginx.*worker" | head -1) -e epoll_wait
# Verify epoll_wait returns events (not spinning)
```

**Mitigation**:
- Keep kernel updated (latest patches for epoll)
- Test under representative load before deployment
- Monitor worker CPU usage (high CPU without throughput = event starvation)

**Severity**: 🔴 **CRITICAL** – Kernel bug

---

## Assumption 2: Clock Source is Monotonic

**Assumption**: System clock (CLOCK_MONOTONIC or equivalent) advances monotonically; no backward jumps.

**Where Relied On**:
- `src/core/ngx_times.c:22` (ngx_time_init)
- `src/event/ngx_event_timer.c:150` (timer expiration)

**Code** (`src/core/ngx_times.c`):
```c
void
ngx_time_update(void)
{
    ngx_cached_time[ngx_time_slot].sec = sec;
    ngx_cached_time[ngx_time_slot].msec = msec;
    
    // Uses gettimeofday() or clock_gettime(CLOCK_MONOTONIC)
    // Assumes time always >= previous time
}
```

**Failure Scenario**:
- NTP adjustment backward (clock skew)
- System time set backward manually
- Timer with expiry T1 never fires (now < T1 after skew)
- Keep-alive connection hangs indefinitely; timeout never triggers

**Impact**: **HIGH** – Connections stuck; resource leak.

**Validation Method**:
```bash
# 1. Check NTP is synchronized
ntpstat
# Expected: "synchronized" (not "unsynchronized")

# 2. Monitor clock jumps
dmesg | grep -i "time jump\|clock skew"

# 3. Test backward skew effect
# Simulate clock jump:
sudo date -s "2024-01-01 10:00:00"
# Verify nginx doesn't hang
curl http://localhost/
# Wait, then:
sudo ntpdate -s ntp.ubuntu.com  # Restore correct time

# 4. Verify monotonic clock in use
strace -e clock_gettime -p $(pgrep -f nginx)
# Should show CLOCK_MONOTONIC, not CLOCK_REALTIME

# 5. Monitor timer queue on production
# Add debug log to ngx_event_timer.c: timer_interval >= 0
```

**Mitigation**:
- Use `CLOCK_MONOTONIC` (doesn't jump backward on NTP adjustment)
- Verify kernel configured for monotonic clock
- Disable manual time adjustments (rely on NTP)
- Monitor for "time jump" kernel messages
- Test graceful degradation: what happens if timer never fires?

**Severity**: 🟠 **HIGH** – Can happen in production (NTP, VM migration).

---

## Assumption 3: File Descriptors Don't Wrap

**Assumption**: File descriptor numbers never wrap around (don't reuse fd 3, 4, ..., then back to 3).

**Where Relied On**:
- `src/core/ngx_connection.c:222` (fd assignment)
- Event multiplexor uses fd as array index

**Code**:
```c
// epoll_ctl stores fd in event.data.fd
// If fd wraps, stale mappings cause confusion
```

**Failure Scenario**:
- Long-running server (100 billion connections over years)
- FD 1001 closed, then reopened as new connection
- Old epoll event for fd 1001 still in queue
- New connection receives stale event intended for old connection
- Data corrupted; protocol violation

**Impact**: **MEDIUM** – Rare; requires extreme uptime and churn.

**Validation Method**:
```bash
# 1. Verify fd range
ulimit -n
# Expected: 65536 (or higher)
# At scale: if server processes 1M conn/sec, 65536 fds cycle every 65ms

# 2. Monitor fd reuse rate
lsof -p $(pgrep -f nginx | head -1) | wc -l
# Compare before/after: should stabilize

# 3. Stress test with explicit FD exhaustion
# Launch many workers, many connections; measure fd churn
```

**Mitigation**:
- Modern systems unlikely to hit 32-bit fd wrap
- If concerned: use `ulimit -n` to track fd usage
- Set high `worker_rlimit_nofile` to enable more fds
- Monitor `ss` or `netstat` for established connection count

**Severity**: 🟡 **MEDIUM** – Mostly theoretical (would require extreme scale + uptime).

---

## Assumption 4: Signals are Async-Signal-Safe

**Assumption**: Signal handlers only call async-signal-safe functions; no malloc, locks, or I/O.

**Where Relied On**:
- `src/os/unix/ngx_process.c:24` (ngx_signal_handler)
- All signals: SIGHUP, SIGUSR1, SIGTERM, SIGCHLD, etc.

**Code** (`src/os/unix/ngx_process.c`):
```c
static void
ngx_signal_handler(int signo)
{
    char           *action;
    ngx_int_t       ignore_sigchld;
    ngx_err_t       err;
    struct rusage   r;
    pid_t           pid;
    
    ignore_sigchld = 0;
    
    switch (signo) {
    
    case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
        ngx_reconfigure = 1;  // ✓ Atomic assignment (safe)
        break;
    
    case ngx_signal_value(NGX_REOPEN_SIGNAL):
        ngx_reopen = 1;  // ✓ Atomic assignment
        break;
    
    case SIGALRM:
        ngx_sigalrm = 1;  // ✓ Atomic assignment
        break;
    
    case SIGCHLD:
        ngx_reap = 1;  // ✓ Atomic assignment
        break;
    }
}
```

**Failure Scenario**:
- Signal handler accidentally calls non-async-safe function (e.g., pthread_mutex_lock, malloc)
- Handler interrupts another thread holding mutex
- Deadlock: signal handler waits for mutex; main thread can't release it
- Worker hangs (deadlock) or crashes (stack corruption)

**Impact**: **HIGH** – Deadlock or corruption.

**Validation Method**:
```bash
# 1. Audit signal handler
grep -n "ngx_signal_handler" src/os/unix/ngx_process.c

# 2. Check for forbidden calls
nm nginx | grep -E "pthread_mutex|malloc|free|fprintf"
# Should not be called from signal handler

# 3. Test SIGTERM delivery under load
# Generate traffic, then: kill -TERM $pid
# Expected: graceful shutdown (not hang or crash)

# 4. Valgrind with thread checking
valgrind --tool=helgrind nginx
# Run with signals; verify no race conditions reported
```

**Mitigation**:
- ✓ nginx correctly uses only atomic assignments in handlers
- Verify no calls to: malloc, pthread_mutex_*, fprintf (lock-based I/O)
- Review any changes to signal handler carefully
- Test signal delivery under load

**Severity**: 🟢 **LOW** – nginx signal handler is correct; risk is in modifications.

---

## Assumption 5: Accept Queue Doesn't Overflow

**Assumption**: Listening socket's accept queue (backlog) doesn't drop connections due to overflow.

**Where Relied On**:
- `src/core/ngx_connection.c:18-29` (ngx_listening_t.backlog)
- Socket listen(2) call with backlog parameter

**Code** (`src/core/ngx_connection.c`):
```c
struct ngx_listening_s {
    int                 backlog;  // listen() backlog parameter
};
```

**Failure Scenario**:
- Accept queue fills (all workers busy, no worker accepting)
- New SYN packets dropped by kernel (not even SYN-ACK sent)
- Client times out; thought server is down
- Asymmetric: server receiving requests but clients never connect

**Impact**: **MEDIUM** – Clients see high latency/timeouts.

**Validation Method**:
```bash
# 1. Check backlog setting
netstat -an | grep LISTEN
# Shows accept queue depth

# 2. Monitor accept queue usage
ss -tan
# Shows "recv-Q" (accept queue occupancy)

# 3. Stress test: saturate event loop
# Generate slow requests (each takes 10 seconds)
# Then launch burst of new connections
# Monitor: do new connections queue or drop?

# 4. Tuning: increase backlog
sysctl net.core.somaxconn
# Default often 128; should be > expected queue depth
```

**Mitigation**:
- Set `listen backlog` directive >= expected concurrent accept queue depth
- Tune `net.core.somaxconn` kernel parameter (global max)
- Monitor `ss -tan` for queue buildup
- Reduce per-worker processing time (keep-alive connections move out quickly)

**Severity**: 🟡 **MEDIUM** – Visible as connection failures under high load.

---

## Assumption 6: Memory Allocations Never Fail Silently

**Assumption**: `malloc()` either succeeds or returns NULL; no out-of-memory kills (OOM).

**Where Relied On**:
- `src/core/ngx_palloc.c` (all allocations checked for NULL)

**Code**:
```c
void *
ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;
    
    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL) {
        return NULL;  // ✓ Caller must check
    }
    
    // ...
    return p;
}
```

**Failure Scenario 1: OOM Killer**:
- Server running out of memory
- Linux OOM killer activates: kills random processes
- SIGKILL sent to nginx worker → no graceful shutdown
- Connections abruptly closed; no response sent

**Failure Scenario 2: Malloc Returns Pointer but Process Later OOM**:
- malloc() succeeds; pointer returned
- Later, page fault on write to allocated memory
- SIGKILL → process dies
- Connections stuck; worker vanishes

**Impact**: **HIGH** – Abrupt worker death.

**Validation Method**:
```bash
# 1. Monitor memory usage
watch -n 1 'free -h; ps aux | grep nginx'

# 2. Stress test memory pressure
# Use `stress-ng` or similar to reduce available memory
stress-ng --vm 1 --vm-bytes 80% --timeout 60s

# 3. Check OOM killer behavior
cat /proc/sys/vm/overcommit_memory
# 0 = heuristic (default)
# 1 = always allow
# 2 = never (prevent OOM killer)
# Recommend 2 or configure memory reserves

# 4. Log allocation failures
grep "alloc:" /var/log/nginx/error.log
```

**Mitigation**:
- **Disable overcommit**: Set `vm.overcommit_memory = 2` (prevent OOM killer)
- **Pre-allocate memory**: Calculate peak working set; ensure available
- **Monitoring**: Track memory growth over time
- **Connection limits**: Set `worker_connections` to limit per-worker memory
- **Worker restart**: Configure reload/restart to release memory periodically

**Severity**: 🔴 **CRITICAL** – OOM killer is uncontrollable.

---

## Assumption 7: Connection Pool Size Matches Actual Usage

**Assumption**: `worker_connections` setting matches realistic concurrent connection count; no starvation.

**Where Relied On**:
- `src/event/ngx_event.c:134-145` (event array allocation)
- `src/core/ngx_cycle.c` (connection array sizing)

**Code** (`src/event/ngx_event.c`):
```c
cycle->connections = ngx_alloc(size * sizeof(ngx_connection_t), cycle->log);
cycle->read_events = ngx_alloc(size * sizeof(ngx_event_t), cycle->log);
cycle->write_events = ngx_alloc(size * sizeof(ngx_event_t), cycle->log);
// size = worker_connections
```

**Failure Scenario**:
- `worker_connections 512` configured
- Peak load reaches 600 concurrent connections
- New connection arrives; no connection slot available
- `ngx_get_connection()` returns NULL
- New connection rejected with 500 error (or no response)

**Impact**: **MEDIUM** – Users rejected; backpressure fails.

**Validation Method**:
```bash
# 1. Baseline load testing
# Ramp up connections until errors appear
# Expected: should match worker_connections

# 2. Monitor active connections
nginx -s stats  # stub_status module
watch -n 1 'curl -s http://localhost/stats | grep active'

# 3. Calculate per-connection memory
# (connections_size + buffers) × worker_processes
# Example: 512 conn × 8KB pool × 4 workers = 16 MB

# 4. Alerts
# active_connections > worker_connections * 0.75 → warning
# active_connections == worker_connections → critical
```

**Mitigation**:
- Profile application to estimate typical/peak connections
- Set `worker_connections` 20-50% above peak
- Monitor active connection count
- Use connection limits at reverse proxy layer (don't rely on nginx alone)

**Severity**: 🟡 **MEDIUM** – Prevents full utilization of capacity.

---

## Assumption 8: Keep-Alive Timeout Doesn't Leak Connections

**Assumption**: Idle keep-alive connections are closed by `keepalive_timeout`; no indefinite hold.

**Where Relied On**:
- `src/http/ngx_http_request.c` (keep-alive timer)
- `src/core/ngx_event_timer.c` (timer expiration)

**Failure Scenario**:
- Client connects but never sends data (or sends very slowly)
- Keep-alive timer expires
- But: buggy timeout calculation or timer never fired (clock skew, see Assumption 2)
- Connection held indefinitely
- Eventually: connection limit reached; new clients rejected

**Impact**: **MEDIUM** – Resource leak; eventual connection starvation.

**Validation Method**:
```bash
# 1. Verify timeout fires
strace -e timer_settime,timer_gettime -p $(pgrep -f nginx)

# 2. Test slow client
# Connect but don't send request
telnet localhost 80
# Wait > keepalive_timeout (default 75s)
# Expected: connection closes (EOF or "Connection reset")

# 3. Monitor connection lifetime
# Add logging: ngx_log_debug("connection closed after %dms", lifetime_ms)

# 4. Check timer heap validity
# Verify no timers with expiry in past
```

**Mitigation**:
- Set `keepalive_timeout` to reasonable value (5-30 seconds typical)
- Monitor connection count distribution (old connections should not accumulate)
- Implement connection limits at reverse proxy level
- Verify Assumption 2 (clock monotonic)

**Severity**: 🟡 **MEDIUM** – Depends on Assumption 2 (clock).

---

## Assumption 9: epoll/kqueue Doesn't Lose Events Under Load

**Assumption**: Event multiplexor accurately reports all I/O readiness under high event rate.

**Where Relied On**:
- All event processing (`ngx_process_events`)

**Failure Scenario**:
- 100,000 concurrent connections
- Burst of events (many sockets become readable simultaneously)
- epoll internal queue overflows
- Some events dropped; workers miss readiness notifications
- Affected sockets starve; connections hang

**Impact**: **MEDIUM** – Sporadic hangs under extreme load.

**Validation Method**:
```bash
# 1. Stress test with high event rate
# Generate many simultaneous reads on many connections
# Monitor latency (should remain stable)

# 2. Kernel version check
# Known issue: older kernels (pre-2.6.28) had epoll bugs
uname -r

# 3. Application-level heartbeat
# Client and server periodically exchange heartbeats
# If heartbeat missed → detect event loss

# 4. Instrumentation
# Count: events_reported vs events_queued
```

**Mitigation**:
- Use latest kernel (epoll fixes in newer versions)
- Set high `worker_rlimit_nofile` to increase event queue size
- Tune epoll buffer size (if supported)
- Test under realistic load before deployment

**Severity**: 🟡 **MEDIUM** – Requires extreme load to trigger.

---

## Assumption 10: NUMA Cache Coherency is Sufficient

**Assumption**: Multi-socket systems maintain cache coherency for accept mutex and atomic counters.

**Where Relied On**:
- `src/event/ngx_event.c:219-224` (accept mutex spinlock)
- `src/event/ngx_event.c:47-48` (atomic counters)

**Failure Scenario (Multi-Socket NUMA System)**:
- 2 sockets, each with 8 cores (16 total)
- Worker 0 spins on accept mutex (Socket 0, L3 cache line)
- Worker 8 spins on same mutex (Socket 1, L3 cache line)
- Cache coherency traffic: Socket 0 L3 → memory → Socket 1 L3
- Latency spike: 100+ µs per mutex operation
- Throughput drops dramatically under contention

**Impact**: **MEDIUM** – Performance degradation on NUMA.

**Validation Method**:
```bash
# 1. Detect NUMA system
lscpu | grep NUMA

# 2. Measure accept mutex latency
# Instrument ngx_shmtx_lock(); time spinlock duration

# 3. Compare single-socket vs multi-socket performance
# Same test, pinned to single socket (numactl --cpunodebind=0)
# vs. allowed to run on both

# 4. Monitor memory bandwidth
perf stat -e LLC-loads,LLC-load-misses
```

**Mitigation**:
- Enable `reuseport` (SO_REUSEPORT) to eliminate accept mutex
- Pin workers to sockets via `worker_cpu_affinity`
- Ensure accept mutex is cache-line aligned (nginx does this)
- Consider thread pools for I/O operations (reduces event load)

**Severity**: 🟡 **MEDIUM** – Performance impact; not correctness issue.

---

## Assumption 11: Disk I/O is Non-Blocking (or Delegated)

**Assumption**: File operations (open, read, stat) never block the event loop indefinitely.

**Where Relied On**:
- `src/core/ngx_open_file_cache.c` (caching to avoid stat)
- `src/http/ngx_http_core_module.c:1800+` (file operations)

**Code**:
```c
// open_file_cache: caches open() and stat() results
// Prevents repeated syscalls for same file
// But: first open() may block!
```

**Failure Scenario**:
- NFS-mounted directory with network latency
- open("/mnt/nfs/file", ...) blocks for 1 second
- Event loop stalled; all connections in that worker frozen
- Other workers still serving traffic (unfair)

**Impact**: **HIGH** – Worker becomes unresponsive.

**Validation Method**:
```bash
# 1. Detect blocking I/O
strace -o /tmp/trace.txt -e openat,stat,fstat -p $(pgrep -f nginx)
# Check: are there long delays between syscall and return?

# 2. Monitor worker CPU/throughput
# Issue slow NFS I/O; measure throughput drop on one worker

# 3. Test with artificial latency
# Use tc (traffic control) to add NFS latency:
tc qdisc add dev eth0 root netem delay 500ms
# Restart nginx; measure impact

# 4. Check open_file_cache effectiveness
# Monitor: cache hits vs misses
```

**Mitigation**:
- Use local storage for static files (not NFS)
- Enable `open_file_cache` (cache stat results)
- Set `open_file_cache_valid` (TTL for cache; default 60s)
- Avoid high-latency filesystems
- Consider async file I/O (not available in standard nginx; possible with patches)

**Severity**: 🟠 **HIGH** – Can occur in production (NFS, slow storage).

---

## Assumption 12: Module Modifications Don't Introduce Race Conditions

**Assumption**: Third-party modules (or custom patches) don't add shared state without synchronization.

**Where Relied On**:
- Module loading: `src/core/ngx_module.c`
- All third-party modules

**Failure Scenario**:
- Custom module adds global counter (no locking)
- All workers increment counter on every request
- Race condition: counter value corrupted
- Statistics incorrect; module logic fails

**Impact**: **HIGH** – Data corruption in module state.

**Validation Method**:
```bash
# 1. Code review: audit all modules for shared mutable state
# grep -n "static.*[a-z_]*\s*=" src/modules/*.c
# For each: verify locked or atomic

# 2. Valgrind with thread checking (if modules use threads)
valgrind --tool=helgrind nginx

# 3. Stress test with modules enabled
# High request rate; verify no crashes or corruption

# 4. Custom linter: check for non-atomic shared state
```

**Mitigation**:
- Code review modules for thread-safety
- Use atomic operations for counters
- Use mutexes for complex shared state
- Test modules under load before deployment
- Prefer read-only shared state (immutable config)

**Severity**: 🔴 **CRITICAL** – Depends on modules used.

---

## Assumption 13: Configuration Syntax is Validated

**Assumption**: Configuration file parsing validates all directives; invalid syntax caught.

**Where Relied On**:
- `src/core/ngx_conf_file.c:158` (ngx_conf_parse)

**Failure Scenario**:
- Typo in config: `worker_processors 8` (typo: not `worker_processes`)
- Parser skips unknown directives (silently ignored)
- Only 1 worker spawned (default; directive never applied)
- Performance degraded; operator doesn't realize

**Impact**: **MEDIUM** – Silent misconfiguration.

**Validation Method**:
```bash
# 1. Enable strict parsing
nginx -t  # test config; should show errors

# 2. Check logs for unknown directives
nginx -s reload
tail /var/log/nginx/error.log | grep -i unknown

# 3. Verify directives applied
# Add test directive; verify logged if misspelled

# 4. Automated config validation
# Script: parse config; verify all directives recognized
```

**Mitigation**:
- Always run `nginx -t` after config changes
- Set `error_log` level to `warn` or `notice` to catch unknown directives
- Use configuration management tool to validate before deployment
- Document all used directives; flag typos in review

**Severity**: 🟡 **MEDIUM** – Caught by `nginx -t`; but easy to miss if skipped.

---

## Assumption 14: Worker Processes Exit Cleanly

**Assumption**: On graceful shutdown (SIGQUIT), workers exit after completing in-flight requests.

**Where Relied On**:
- `src/os/unix/ngx_process_cycle.c:699-749` (worker loop)

**Code** (`ngx_process_cycle.c:728-741`):
```c
if (ngx_quit) {
    ngx_quit = 0;
    ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                  "gracefully shutting down");
    
    if (!ngx_exiting) {
        ngx_exiting = 1;
        ngx_set_shutdown_timer(cycle);         // Set timer (default: 600s, configurable)
        ngx_close_listening_sockets(cycle);    // Stop accepting
        ngx_close_idle_connections(cycle);     // Close idle connections
    }
}
```

**Failure Scenario**:
- Worker processing long-running request (e.g., file upload)
- SIGQUIT sent; worker enters graceful shutdown
- `ngx_exiting = 1`; listening socket closed
- But: slow request takes 10 minutes
- Shutdown timer fires (600s default): worker force-exits
- Request interrupted; no response sent to client

**Impact**: **MEDIUM** – In-flight requests aborted.

**Validation Method**:
```bash
# 1. Test graceful shutdown
# Start nginx; generate slow request (e.g., 30s timeout)
sleep 30 | curl -T /dev/stdin http://localhost/upload

# In another terminal:
nginx -s quit  # graceful shutdown

# Observe: does request complete or hang?

# 2. Monitor shutdown logs
tail -f /var/log/nginx/error.log
# Should log: "gracefully shutting down"
# Then: workers exiting

# 3. Tune shutdown timer
worker_shutdown_timeout 30s;  # default: nginx.conf
```

**Mitigation**:
- Set `worker_shutdown_timeout` appropriately (consider max request duration)
- For long-running operations: implement timeout or background processing
- Monitor shutdown duration; alert if exceeds expected time
- Document upgrade/restart procedures

**Severity**: 🟡 **MEDIUM** – Expected behavior; tune as needed.

---

## Assumption 15: Configured Limits Prevent Resource Exhaustion

**Assumption**: Tuning directives (worker_connections, client_max_body_size, etc.) provide sufficient backpressure.

**Where Relied On**:
- Multiple directives; see BUFFER_MANAGEMENT.md

**Failure Scenario**:
- `worker_connections 512` (default)
- `client_max_body_size 1m` (default)
- Attack: 512 connections × 1 MB each = 512 MB memory per worker
- 4 workers × 512 MB = 2 GB total
- Server runs out of memory → OOM killer activates

**Impact**: **HIGH** – Resource exhaustion.

**Validation Method**:
```bash
# 1. Calculate worst-case memory usage
per_worker = (worker_connections * pool_size) + (max_body_size * concurrent_uploads)
total = per_worker * worker_processes + OS_overhead
# Example: (512 * 8KB) + (1MB * 10) = 14 MB per worker; 60 MB total (reasonable)

# 2. Stress test with limits
# Connect worker_connections times; upload client_max_body_size each
# Measure memory; should stay below prediction

# 3. Monitor under production load
watch -n 1 'ps aux | grep nginx; free -h'
```

**Mitigation**:
- Calculate per-worker memory footprint
- Set limits below available RAM with safety margin (e.g., 50% of RAM)
- Monitor actual memory usage; alert if exceeds 80%
- Implement rate limiting at reverse proxy layer
- Use request timeouts to prevent slow-read DoS

**Severity**: 🟠 **HIGH** – Depends on configuration and workload.

---

## Risk Matrix

| # | Assumption | Severity | Likelihood | Mitigation Effort |
|---|-----------|----------|------------|------------------|
| 1 | Event multiplexor correct | 🔴 CRITICAL | Low | Medium (kernel) |
| 2 | Clock monotonic | 🟠 HIGH | Medium | Low (NTP config) |
| 3 | FD no wrap | 🟡 MEDIUM | Very Low | None (system limit) |
| 4 | Signals safe | 🟢 LOW | Low | Low (code review) |
| 5 | Accept queue OK | 🟡 MEDIUM | Medium | Low (tune backlog) |
| 6 | Memory safe | 🔴 CRITICAL | Medium | Medium (config) |
| 7 | Connection pool sized | 🟡 MEDIUM | Medium | Low (monitoring) |
| 8 | Keep-alive timeout works | 🟡 MEDIUM | Low | Low (verify clock) |
| 9 | epoll OK under load | 🟡 MEDIUM | Low | Low (kernel update) |
| 10 | NUMA coherency | 🟡 MEDIUM | Low | Medium (reuseport) |
| 11 | Disk I/O non-blocking | 🟠 HIGH | Medium | Low (local storage) |
| 12 | Modules thread-safe | 🔴 CRITICAL | Medium | High (code review) |
| 13 | Config validated | 🟡 MEDIUM | Medium | Low (script) |
| 14 | Workers exit cleanly | 🟡 MEDIUM | Low | Low (tune timeout) |
| 15 | Limits prevent exhaustion | 🟠 HIGH | Medium | Medium (monitoring) |

---

## Validation Checklist

### Pre-Deployment

- [ ] Kernel version tested (no known epoll bugs)
- [ ] NTP synchronized; `CLOCK_MONOTONIC` in use
- [ ] Memory available for worst-case connection set
- [ ] OOM killer disabled (or configured appropriately)
- [ ] Backlog and file descriptor limits tuned
- [ ] Configuration validated (`nginx -t`)
- [ ] Modules audited for thread-safety
- [ ] Storage local and low-latency (no NFS)

### Ongoing Monitoring

- [ ] Connection count < 90% of `worker_connections`
- [ ] Memory usage stable (no gradual growth)
- [ ] Worker CPU time even across workers
- [ ] No timeout/retry spikes in access logs
- [ ] Clock skew alerts monitored
- [ ] OOM killer never triggered (`dmesg | grep OOM`)

### Incident Response

- [ ] Graceful shutdown procedures documented
- [ ] Metrics for "unhealthy worker" defined
- [ ] Automated worker restart/reload capability
- [ ] RCA process for crashes/hangs

---

## Conclusion

These 15 assumptions are mostly **validated by nginx's design**, but depend on:
1. Correct OS configuration (NTP, OOM, limits)
2. Appropriate nginx configuration (connection pool size, timeouts)
3. Operational discipline (monitoring, configuration review)
4. Third-party modules (custom modules must be reviewed)

**The biggest risks are in assumptions #2 (clock), #6 (memory), #11 (disk I/O), and #12 (modules)**. Focus hardening efforts there.

---

**References**:
- POSIX Signal Safety: https://pubs.opengroup.org/onlinepubs/9699919799/functions/V2_chap02.html
- Linux epoll man page: `man 7 epoll`
- NUMA Performance: "NUMA Aware Data Structures and Algorithms for Multicore and Hybrid Machines" (Afek et al.)

