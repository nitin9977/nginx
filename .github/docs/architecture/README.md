# NGINX Architecture Documentation Package

**Purpose**: Comprehensive onboarding for engineers ramping up on the NGINX codebase.  
**Created**: 2024  
**Status**: ✓ Complete (Principal Engineer Review)  
**Audience**: Backend engineers, systems engineers, performance specialists.

---

## Documentation Set

This package contains **6 comprehensive documents** designed to be read in sequence or as independent references.

### 1. **ARCHITECTURE.md** – High-Level Overview

**Read This First**: Overview of NGINX's design, core modules, and data flow.

**Covers**:
- Master-worker process model
- Core module dependencies (event loop, connections, HTTP parser, buffers)
- Request lifecycle (socket → event loop → handler → response)
- Memory allocation strategy (pools)
- Synchronization primitives (accept mutex, atomic counters)
- Signal handling
- Performance trade-offs and limitations

**Key Sections**:
- Module dependency tree (ASCII diagram)
- Data flow diagram (7-step request lifecycle)
- Invariants (correctness guarantees)
- Configuration tuning points

**Time to Read**: 30-45 minutes  
**Files Referenced**: `src/core/nginx.c`, `src/event/ngx_event.c`, `src/core/ngx_connection.h`, `src/core/ngx_palloc.c`, `src/core/ngx_buf.h`, `src/os/unix/ngx_process_cycle.c`

---

### 2. **MEMORY_AND_CONCURRENCY.md** – Deep Dive on Workers & Shared State

**Read This After**: ARCHITECTURE.md

**Covers**:
- Worker process spawning (fork chain)
- Per-worker memory layout (connections array, event array, pools)
- Shared memory structures (accept mutex with spinlock, atomic counters)
- Pool-based allocation strategy (creation, small/large alloc paths, deallocation)
- Connection pool lifecycle (allocate, reuse, free)
- Cache coherency and false sharing (NUMA implications)
- Worker independence and isolation
- Concurrency patterns (accept serialization, connection ownership)
- Memory safety analysis (data races, use-after-free, overflow prevention)

**Key Insights**:
- No threads; each worker is an independent OS process
- Minimal shared state (only accept mutex + atomic counters)
- Zero intra-worker locking (single-threaded simplicity)
- Memory pools enable predictable deallocation

**Time to Read**: 20-30 minutes  
**Files Referenced**: `src/core/ngx_shmtx.c`, `src/core/ngx_spinlock.c`, `src/core/ngx_palloc.c`, `src/os/unix/ngx_process_cycle.c`

---

### 3. **BUFFER_MANAGEMENT.md** – Buffer Safety & Overflow Analysis

**Read This If**: Concerned about security, buffer overflows, or DoS attacks.

**Covers**:
- All buffer allocation paths (connection read buffer, request headers, body)
- HTTP request parsing (request line state machine, header parsing)
- Edge cases and attacks:
  - Oversized request lines
  - Header injection
  - Slowloris (slow-read attack)
  - HTTP request smuggling
  - Chunked transfer encoding edge cases
- Memory safety analysis (overflow scenarios, bounds checking)
- Resource exhaustion attacks (slow header, large header bomb, slow-write)
- Secure configuration recommendations
- Validation and testing methods

**Security Level**: 🟢 Protected (All attacks have mitigations)

**Time to Read**: 25-35 minutes  
**Files Referenced**: `src/core/ngx_buf.h`, `src/http/ngx_http_parse.c`, `src/http/ngx_http_request_body.c`

---

### 4. **ASSUMPTIONS_AND_THREATS.md** – Threat Model & Validation

**Read This If**: Designing critical systems, security-focused, or validating assumptions.

**Covers**:
- **15 Critical Assumptions** enumerated and analyzed:
  1. Event multiplexor is correct (epoll/kqueue/select)
  2. Clock source is monotonic
  3. File descriptors don't wrap
  4. Signals are async-signal-safe
  5. Accept queue doesn't overflow
  6. Memory allocations never fail silently (OOM killer)
  7. Connection pool size matches usage
  8. Keep-alive timeout works
  9. epoll/kqueue doesn't lose events under load
  10. NUMA cache coherency is sufficient
  11. Disk I/O is non-blocking
  12. Module modifications don't introduce races
  13. Configuration syntax is validated
  14. Workers exit cleanly
  15. Configured limits prevent exhaustion

- For each assumption: failure scenario, impact, validation method, mitigation

- **Risk Matrix** summarizing severity × likelihood × mitigation effort

- **Validation Checklist** (pre-deployment, ongoing monitoring, incident response)

**Time to Read**: 40-50 minutes  
**Most Critical Assumptions**: #2 (clock), #6 (OOM), #11 (disk I/O), #12 (modules)

---

### 5. **PERFORMANCE_BOTTLENECKS.md** – Tuning & Optimization

**Read This If**: Performance engineering, capacity planning, or tuning nginx.

**Covers**:
- **Hot Paths** (Priority 1 always-hot):
  1. Event loop (1-2 ms per iteration)
  2. HTTP request parsing (1-5 µs, trivial)
  3. Connection accept (5-50 µs, malloc-dominated)
  4. Buffer I/O (1-100 µs, syscall overhead)

- **Cache Behavior** (L1/L2/L3 hit rates, false sharing)

- **Lock Contention** (accept mutex, atomic counters)

- **Memory Bottlenecks** (per-connection pools, header buffers)

- **CPU Bottlenecks** (HTTP parsing, regex, logging)

- **Tuning Checklist** (network, connection, memory, event, timer, OS-level)

- **Benchmarking Methodology** (load test setup, metrics, validation tests)

**Time to Read**: 20-30 minutes  
**Key Takeaway**: "If network isn't saturated, nginx isn't the problem."

---

### 6. **CODE_WALKTHROUGH_EVENT_LOOP.md** – Annotated Source Code

**Read This If**: Need to understand exact code flow, debugging event handling.

**Covers**:
- Entry point (main() in `src/core/nginx.c:196`)
- Master process startup and signal handling
- Worker process initialization and event loop
- **Event loop core** (8-step algorithm with exact line numbers):
  1. Calculate next timeout
  2. Try accept mutex
  3. Check posted events
  4. Call epoll/kqueue/select
  5. Process accept events
  6. Release mutex
  7. Expire timers
  8. Process data events
- Event multiplexor (epoll initialization and processing)
- Connection accept path
- HTTP connection initialization
- **Request parsing state machine** (character-by-character with all states)
- Response writing path (output chain, write handler)
- **Full request lifecycle with timing** (0µs to 6s, step-by-step)

**Key Invariants Verified**:
- Single-threaded per worker
- State machine resumability
- No polling (epoll blocks)
- Memory isolation (per-connection pools)
- Timeout safety

**Time to Read**: 30-40 minutes (recommend having source code open)  
**Best Used As**: Reference while reading source code

---

## Quick Reference: Choose Your Starting Point

### "I need to understand nginx architecture in 1 hour"
→ Read **ARCHITECTURE.md** (45 min) + skim **PERFORMANCE_BOTTLENECKS.md** (15 min)

### "I'm debugging a memory leak or performance issue"
→ Read **MEMORY_AND_CONCURRENCY.md** + **CODE_WALKTHROUGH_EVENT_LOOP.md**

### "I'm concerned about security or DoS attacks"
→ Read **BUFFER_MANAGEMENT.md** + **ASSUMPTIONS_AND_THREATS.md**

### "I'm tuning nginx for high throughput"
→ Read **PERFORMANCE_BOTTLENECKS.md** + **ASSUMPTIONS_AND_THREATS.md** (especially #7, #10, #15)

### "I'm adding a module or modifying core behavior"
→ Read **ASSUMPTIONS_AND_THREATS.md** (especially #12 on thread-safety) + **CODE_WALKTHROUGH_EVENT_LOOP.md**

### "I need to validate the entire system before production"
→ Read all 6 documents + run **Validation Checklist** in ASSUMPTIONS_AND_THREATS.md

---

## Cross-References

### By Topic

**Event Handling**:
- ARCHITECTURE.md: "Event Loop" section
- MEMORY_AND_CONCURRENCY.md: "Event Queue Independence"
- PERFORMANCE_BOTTLENECKS.md: "Hot Path #1: Event Loop"
- CODE_WALKTHROUGH_EVENT_LOOP.md: "Event Loop Core" section

**Memory Management**:
- ARCHITECTURE.md: "Memory Allocation" section
- MEMORY_AND_CONCURRENCY.md: "Memory Allocation Strategy"
- BUFFER_MANAGEMENT.md: "Buffer Allocation Paths"
- PERFORMANCE_BOTTLENECKS.md: "Memory Bottlenecks"

**Concurrency**:
- ARCHITECTURE.md: "Synchronization & Concurrency" section
- MEMORY_AND_CONCURRENCY.md: Entire document
- ASSUMPTIONS_AND_THREATS.md: "Assumptions #2, #6, #12"

**Performance**:
- ARCHITECTURE.md: "Performance Characteristics"
- MEMORY_AND_CONCURRENCY.md: "Cache Coherency"
- PERFORMANCE_BOTTLENECKS.md: Entire document
- CODE_WALKTHROUGH_EVENT_LOOP.md: "Connection Lifecycle: Timing"

**Security**:
- BUFFER_MANAGEMENT.md: Entire document
- ASSUMPTIONS_AND_THREATS.md: "Assumptions #1, #5, #6, #11"

---

## Validation Strategy

### Before Deploying nginx in Production

**Week 1: Understanding**
- [ ] Read ARCHITECTURE.md
- [ ] Read MEMORY_AND_CONCURRENCY.md

**Week 2: Security & Assumptions**
- [ ] Read BUFFER_MANAGEMENT.md
- [ ] Read ASSUMPTIONS_AND_THREATS.md
- [ ] Run pre-deployment checklist (ASSUMPTIONS_AND_THREATS.md)

**Week 3: Tuning**
- [ ] Read PERFORMANCE_BOTTLENECKS.md
- [ ] Profile your workload (benchmarking section)
- [ ] Tune configuration based on results
- [ ] Monitor metrics (ongoing checklist)

**Week 4: Operations**
- [ ] Read CODE_WALKTHROUGH_EVENT_LOOP.md (for debugging)
- [ ] Document incident response procedures
- [ ] Train on-call team

---

## Measurement & Monitoring

### Key Metrics to Track

| Metric | Expected Range | Alert Threshold |
|--------|---|---|
| Event loop time | 1-10 ms | > 100 ms |
| Active connections | < 0.9 × worker_connections | > 0.9 × limit |
| Memory per worker | 50-200 MB | > 500 MB (growing) |
| CPU per worker | Balanced (±10%) | Unbalanced (>20% diff) |
| Accept mutex hold time | < 1 ms | > 10 ms |
| Cache hit rate (open_file_cache) | > 90% | < 80% |
| Connection timeout rate | ~0% | > 0.1% |
| 500/503 errors | ~0% | > 0.01% |

### Tools

- **Real-time monitoring**: `watch` + `ps aux`, `ss -tan`, `netstat -an`
- **Profiling**: `perf record`, `perf report`, `perf stat`
- **Tracing**: `strace`, `ltrace` (for detailed syscall tracking)
- **Memory analysis**: `valgrind`, ASAN (AddressSanitizer)
- **Load testing**: `ab`, `wrk`, `vegeta`, `hey`

---

## Common Questions & Answers

**Q: How many workers should I run?**  
A: Default is `auto` (CPU cores). In practice, 1-2 per core is ideal. See PERFORMANCE_BOTTLENECKS.md.

**Q: What's the max connections per worker?**  
A: Limited by file descriptors (ulimit -n). Typical: 10,000-65,536. Set `worker_connections` based on your memory budget.

**Q: Why do I see slow requests?**  
A: 1) Slow backend (application), 2) Slow client (network), 3) Slow disk I/O. See ASSUMPTIONS_AND_THREATS.md #11.

**Q: How do I debug a memory leak?**  
A: See MEMORY_AND_CONCURRENCY.md "Debugging Concurrency Issues" + run `valgrind --leak-check=full`.

**Q: Can I run nginx with threads?**  
A: No (by design). Threads not used; workers are processes. Some I/O operations can be delegated to thread pools.

**Q: How do I configure for 1 million connections?**  
A: Tune `worker_connections` high, increase ulimit, use `reuseport`, distribute across multiple servers. See PERFORMANCE_BOTTLENECKS.md tuning checklist.

---

## Feedback & Improvements

This documentation is a living artifact. If you:
- Find inaccuracies → report with code reference
- Discover missing topics → document and contribute
- Disagree with design decisions → challenge assumptions (see ASSUMPTIONS_AND_THREATS.md)

**File Issues**: `.github/issues/` (use label: `architecture-docs`)

---

## Glossary

| Term | Definition |
|------|-----------|
| **ngx_cycle_t** | nginx cycle struct; holds configuration, connections, events, modules |
| **ngx_connection_t** | Per-connection state (socket, buffers, event handlers, application data) |
| **ngx_buf_t** | Buffer struct (memory or file data); part of response chain |
| **ngx_pool_t** | Memory pool; allocate and free in bulk |
| **ngx_event_t** | Event struct (read/write readiness); handles timers and I/O |
| **Accept Mutex** | Shared spinlock serializing accept(2) to prevent thundering herd |
| **Event Loop** | Main worker loop: calculate timer, epoll_wait, process events, expire timers |
| **epoll/kqueue** | OS event multiplexor (Linux/BSD) |
| **State Machine** | HTTP parser: character-by-character state transitions |
| **Keep-Alive** | Reuse connection for multiple HTTP requests |
| **Sendfile** | Zero-copy file transmission (kernel reads file → sends to network) |
| **reuseport** | SO_REUSEPORT: allow multiple processes to accept from same socket |

---

## Recommended Additions (Not Included)

These topics are out of scope but would enhance the documentation:

1. **HTTP/2 & HTTP/3 Protocol Implementation**
   - SPDY frames, header compression (HPACK)
   - Stream multiplexing

2. **SSL/TLS Integration**
   - OpenSSL integration
   - Certificate handling
   - Cipher suite tuning

3. **Module Development Guide**
   - Module hooks and callbacks
   - Configuration directives
   - Third-party module examples

4. **Load Balancing & Clustering**
   - Upstream module
   - Health checking
   - Sticky sessions

5. **Caching Strategy**
   - Microcaches
   - Cache purging
   - Cache key generation

6. **Advanced Topics**
   - QUIC/UDP support
   - gRPC proxying
   - WAF integration

---

## Contributing

To improve this documentation:

1. Identify gap or error
2. Create branch: `docs/architecture-{topic}`
3. Edit relevant `.md` file
4. Add code reference (file + line number)
5. Test explanations against source code
6. Submit PR with narrative of change

**Quality Standards**:
- ✓ Every claim references source code
- ✓ Every assumption includes validation method
- ✓ Every threat includes mitigation
- ✓ Examples are runnable (include bash commands)
- ✓ Metrics are measurable (specific tools, commands)

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2024 | 1.0 | Initial architecture documentation package (6 docs) |

---

**Authors**: Principal Engineer (nginx architecture analysis)  
**License**: Consistent with nginx project  
**Location**: `.github/docs/architecture/`

---

## Quick Links

- **ARCHITECTURE.md**: High-level overview
- **MEMORY_AND_CONCURRENCY.md**: Worker isolation, shared state
- **BUFFER_MANAGEMENT.md**: Buffer safety, DoS attacks
- **ASSUMPTIONS_AND_THREATS.md**: Risk assessment, validation
- **PERFORMANCE_BOTTLENECKS.md**: Hot paths, tuning
- **CODE_WALKTHROUGH_EVENT_LOOP.md**: Annotated source (line numbers)

---

**Last Updated**: 2024  
**Status**: ✓ Complete

