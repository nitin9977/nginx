# Debug Playbook — nginx Issue Diagnosis

Organized by symptom. For each: likely root causes, which files to check, and verification steps.

---

## Crash / Segfault

### Symptom: worker process crashes with SIGSEGV

**Most likely causes** (in order):

1. **Use-after-free on connection/pool**
   - Connection closed but handler still references `c->data`
   - Pool destroyed but pointer still dereferenced
   - Check: `c->destroyed` flag, pool cleanup handler ordering
   - Files: `src/core/ngx_connection.c` (`ngx_close_connection`), module's close handler

2. **Stale event firing on reused connection slot**
   - epoll returns event for old connection, slot reused for new one
   - Check: `instance` bit mismatch in `ngx_epoll_process_events()`
   - Files: `src/event/modules/ngx_epoll_module.c`, `src/core/ngx_connection.c` (`ngx_get_connection`)

3. **NULL dereference on config context**
   - Module accessing `ctx[module.ctx_index]` before allocation
   - Check: module init order, config inheritance chain
   - Files: module's `create_*_conf`, `merge_*_conf`

4. **Buffer overrun in parser**
   - Malformed request exceeds buffer without bounds check
   - Check: `ngx_http_parse_request_line()` state transitions
   - Files: `src/http/ngx_http_parse.c`

**Verification**: Run with `--with-debug`, enable `error_log stderr debug;`, check core dump backtrace

---

## Memory Leak

### Symptom: worker RSS grows over time

**Most likely causes**:

1. **Pool not destroyed on error path**
   - Early return skips `ngx_destroy_pool()`
   - Check: all error branches in request/session handlers
   - Files: protocol-specific finalize functions

2. **Large allocation not freed before pool destroy**
   - `ngx_palloc_large()` allocations are freed by pool destroy, but if pool is long-lived (cycle pool), they accumulate
   - Check: allocations against `cycle->pool` that should be per-request

3. **Connection leak (fd not closed)**
   - `ngx_close_connection()` not called on error path
   - Check: `cycle->free_connection_n` — if it decreases over time, connections are leaking
   - Files: `src/core/ngx_connection.c`

4. **Shared memory slab leak**
   - Slab allocations without corresponding frees
   - Check: `ngx_slab_alloc()` / `ngx_slab_free()` pairing
   - Files: `src/core/ngx_slab.c`

**Verification**: ASAN build (`--with-cc-opt="-fsanitize=address"`), valgrind with `--leak-check=full`

---

## Connection Issues

### Symptom: connections hang or timeout unexpectedly

**Check in order**:

1. **Free connection pool exhausted**
   - `cycle->free_connection_n == 0`
   - Fix: increase `worker_connections`, check for connection leaks
   - Verify: `ngx_stat_active` vs `worker_connections`

2. **Timer not armed or not cleared**
   - Read timer not set → connection hangs forever
   - Timer not cleared after success → premature timeout
   - Files: check `ngx_add_timer()` / `ngx_del_timer()` pairing in handler

3. **Event not registered with kernel**
   - `ngx_handle_read_event()` / `ngx_handle_write_event()` not called after handler returns
   - Files: protocol handler code

4. **Accept mutex starvation**
   - One worker holds accept mutex too long
   - Check: `accept_mutex_delay` setting, worker load balance
   - Files: `src/event/ngx_event.c` (`ngx_process_events_and_timers`)

5. **Upstream connection failure**
   - Backend unreachable, DNS failure, connection refused
   - Files: `src/http/ngx_http_upstream.c`, `src/event/ngx_event_connect.c`

### Symptom: "too many open files" errors

- Check `worker_rlimit_nofile` directive
- Check `worker_connections` (each connection = 2 fds for proxy)
- Verify: `ulimit -n` in worker context
- Files: `src/os/unix/ngx_posix_init.c` (`ngx_max_sockets`)

---

## Config Reload Failures

### Symptom: SIGHUP doesn't apply new config

**Check in order**:

1. **Config parse error**
   - `ngx_init_cycle()` failed, old cycle retained
   - Check error log for `[emerg]` messages during parse
   - Files: `src/core/ngx_cycle.c` (`ngx_init_cycle`)

2. **Shared memory zone size mismatch**
   - New config changes zone size → zone cannot be reused
   - Check: `ngx_shared_memory_add()` in `src/core/ngx_cycle.c`

3. **Listening socket conflict**
   - New config can't bind to port (already in use by old worker)
   - Check: `ngx_open_listening_sockets()` errors

4. **Module init_module failure**
   - Module's `init_module` callback returns `NGX_ERROR`
   - Check: module-specific init code

**Verification**: Send `kill -HUP <master_pid>`, watch error log

---

## Signal Handling Issues

### Symptom: graceful shutdown hangs

1. **Workers not receiving signal**
   - Master sends via channel, not signal — check channel fd is open
   - Files: `src/os/unix/ngx_process_cycle.c` (`ngx_signal_worker_processes`)

2. **Worker stuck in event loop**
   - Long-running operation blocks `ngx_process_events_and_timers()`
   - Check: signal flags are checked per iteration, but blocked syscalls delay check

3. **Connections not draining**
   - Keepalive connections hold workers alive
   - Check: `ngx_exiting` flag, reusable connection queue drain
   - Files: `src/core/ngx_connection.c` (`ngx_drain_connections`)

### Symptom: binary upgrade fails

1. **Old master can't exec new binary**
   - Check: `ngx_exec_new_binary()` in `src/os/unix/ngx_process_cycle.c`
   - Verify: new binary path, permissions, NGINX env variable for fd inheritance

---

## Event Loop Issues

### Symptom: high latency spikes

1. **Blocking operation in event loop**
   - DNS resolution, file I/O, or heavy computation blocking worker
   - Fix: use async resolver, AIO for disk, limit computation per event

2. **Timer resolution too coarse**
   - Default timer update only on `epoll_wait` return
   - Fix: `timer_resolution` directive forces periodic `SIGALRM`
   - Files: `src/event/ngx_event.c`

3. **Too many events in single poll**
   - `multi_accept on` with burst of connections
   - Check: event batch processing time

4. **Posted event queue backlog**
   - Accept events processed before data events
   - Check: `ngx_posted_accept_events` vs `ngx_posted_events` queue lengths

---

## Phase Engine Issues (HTTP)

### Symptom: request stuck in phase processing

1. **Handler returns wrong code**
   - `NGX_DECLINED` vs `NGX_OK` vs `NGX_DONE` semantics differ per phase
   - Check: phase checker in `src/http/ngx_http_core_module.c`

2. **Content handler not set**
   - No module claimed `r->content_handler`
   - Results in 404 or 500 depending on location config

3. **Subrequest reference count leak**
   - `r->count` never reaches 0, request never finalizes
   - Check: `ngx_http_finalize_request()` reference count logic

4. **Filter chain ordering**
   - Body filter modifies or drops data unexpectedly
   - Check: filter registration order in `postconfiguration`

---

## Phase Engine Issues (Stream)

### Symptom: stream session stuck

1. **Preread handler returns NGX_AGAIN but timer not armed**
   - Session waits forever for data
   - Check: `ngx_stream_core_preread_phase` timer logic

2. **Content handler missing**
   - `cscf->handler == NULL` → internal server error
   - Check: `proxy_pass` or `return` directive in server block

3. **Upstream connection failure**
   - Backend refuses or times out
   - Check: upstream module peer selection, `ngx_stream_proxy_module.c`

---

## Performance Issues

### Symptom: throughput below expectations

1. **Accept mutex contention** — disable with `accept_mutex off;` if using `reuseport`
2. **sendfile disabled** — enable `sendfile on;` for static file serving
3. **TCP options** — enable `tcp_nopush on; tcp_nodelay on;`
4. **Worker count** — set `worker_processes auto;` to match CPU cores
5. **Connection limits** — increase `worker_connections` to match expected concurrency
6. **Buffer sizing** — tune `proxy_buffer_size`, `proxy_buffers`, `client_body_buffer_size`
7. **Keepalive** — enable `keepalive` on upstream blocks for connection reuse

**Measurement**: `stub_status` module, `ngx_http_stub_status_module` for active/waiting/reading/writing counts

---

## Quick Diagnostic Commands

```bash
# Check master/worker PIDs
ps aux | grep nginx

# Send signals
kill -HUP <master_pid>      # reload config
kill -USR1 <master_pid>     # reopen logs
kill -QUIT <master_pid>     # graceful shutdown
kill -USR2 <master_pid>     # binary upgrade

# Check open fds per worker
ls -la /proc/<worker_pid>/fd | wc -l

# Check connection count
curl http://localhost/stub_status

# Enable debug logging (requires --with-debug)
error_log /var/log/nginx/debug.log debug;
```
