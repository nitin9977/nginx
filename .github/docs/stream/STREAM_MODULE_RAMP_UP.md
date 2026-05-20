# NGINX Stream Module Ramp-Up Guide

**Audience**: Engineers onboarding to `src/stream` internals  
**Scope**: Architecture, integration, and walkthrough for representative stream session flows  
**Date**: 2026-05-19

---

## 1. Architecture and Design of `src/stream`

### 1.1 What `src/stream` does

`src/stream` implements TCP/UDP layer-4 proxy and policy processing.

It owns:

- `stream {}` config parsing and module context setup
- Server/listen/address/name optimization for stream listeners
- Per-connection stream session lifecycle
- Phase engine execution for preread/access/content/log stages
- Upstream proxy/load-balancing support for stream traffic
- Variable framework for stream-level scripting

### 1.2 Design model

The stream subsystem mirrors a phase-pipeline architecture:

1. Build phase graph at config time
2. Bind listener metadata to runtime address contexts
3. For each new connection/session, execute phase engine until terminal state

Core files:

- `src/stream/ngx_stream.c` — block parser and phase/listener graph setup
- `src/stream/ngx_stream_handler.c` — runtime session bootstrap and close flow
- `src/stream/ngx_stream_core_module.c` — phase engine, checkers, and core phase directives

### 1.3 Key design objects

#### `ngx_stream_session_s` (`src/stream/ngx_stream.h`)

Per-connection session state:

- `signature`: magic `0x4d525453` (`"STRM"`) — type-safety check
- `connection`: back-pointer to `ngx_connection_t`
- `received`: total bytes received from client (`off_t`)
- `start_sec` / `start_msec`: session start time for duration tracking
- `log_handler`: custom log formatter callback
- `ctx` / `main_conf` / `srv_conf`: per-module context and config arrays
- `virtual_names`: pointer to `ngx_stream_virtual_names_t` for SNI-based server selection
- `upstream`: `ngx_stream_upstream_t *` — upstream connection state (proxy module)
- `upstream_states`: array of `ngx_stream_upstream_state_t` for upstream attempt history
- `variables`: pre-allocated variable value array
- `phase_handler`: current index into `phase_engine.handlers[]`
- `status`: final session status code (200, 400, 403, 500, 502, 503)

**Bit-field flags**:

- `ssl:1`: SSL/TLS active
- `stat_processing:1`: stub status tracking flag
- `health_check:1`: connection is a health check (not client traffic)
- `limit_conn_status:2`: limit_conn module decision state

#### Phase system (`src/stream/ngx_stream.h`)

**Phase enum** (`ngx_stream_phases` — 7 phases):

| Phase | Purpose |
|-------|---------|
| `NGX_STREAM_POST_ACCEPT_PHASE` | Post-accept processing (realip, limit_conn) |
| `NGX_STREAM_PREACCESS_PHASE` | Pre-access checks |
| `NGX_STREAM_ACCESS_PHASE` | Access control (allow/deny) |
| `NGX_STREAM_SSL_PHASE` | SSL/TLS handshake |
| `NGX_STREAM_PREREAD_PHASE` | Read initial data for protocol detection (ssl_preread) |
| `NGX_STREAM_CONTENT_PHASE` | Content generation/proxy (exactly one handler) |
| `NGX_STREAM_LOG_PHASE` | Logging (runs on finalize, not in phase engine) |

**`ngx_stream_phase_handler_s`**:

- `checker`: phase-specific checker function (drives return code interpretation)
- `handler`: the actual phase handler (module code)
- `next`: index of first handler in the next phase (for `NGX_OK` skip-to-next-phase)

**Phase engine semantics** — return codes from `checker()`:

- `NGX_OK` → stop the phase loop (handler took control or finalized)
- `NGX_AGAIN` → continue to next handler in `phase_engine.handlers[]`

**Handler return codes** (interpreted by checkers):

- `NGX_OK` → advance to `ph->next` (skip remaining handlers in current phase)
- `NGX_DECLINED` → advance to `s->phase_handler++` (try next handler in same phase)
- `NGX_AGAIN` / `NGX_DONE` → handler will resume later (async wait)
- `NGX_ERROR` / stream status codes → finalize session

#### `ngx_stream_core_main_conf_t` (`src/stream/ngx_stream.h`)

Main config context:

- `servers`: array of `ngx_stream_core_srv_conf_t`
- `phase_engine`: compiled handler array for runtime execution
- `variables` / `variables_hash`: stream variable framework
- `ports`: array of `ngx_stream_conf_port_t` — listener binding data
- `phases[7]`: per-phase handler arrays (assembled at postconfiguration, compiled into `phase_engine`)

#### `ngx_stream_core_srv_conf_t` (`src/stream/ngx_stream.h`)

Per-server config:

- `server_names`: virtual server name array (for SNI matching)
- `handler`: content phase handler function (`ngx_stream_content_handler_pt`)
- `ctx`: config context for this server block
- `server_name`: default server name
- `tcp_nodelay`: TCP_NODELAY flag
- `preread_buffer_size`: buffer for preread phase data
- `preread_timeout`: timeout for preread phase
- `proxy_protocol_timeout`: PROXY protocol header timeout
- `resolver` / `resolver_timeout`: DNS resolution for upstream
- `error_log`: server-specific error log

### 1.4 Event-driven state transitions

Read handler pointers move between:

- `ngx_stream_proxy_protocol_handler` — PROXY protocol preread
- `ngx_stream_session_handler` — generic session handler (enters phase engine)
- Upstream/content-specific handlers (after content phase takes control)

Asynchronous waiting uses read events + timers, then resumes through the phase engine via `ngx_stream_session_handler`.

---

## 2. Integration and Dependencies

### 2.1 Integration with `src/event`

Event layer accepts listener sockets and calls `ngx_stream_init_connection()`.

Stream then:

- Sets initial read handler (`ngx_stream_session_handler` or `ngx_stream_proxy_protocol_handler`)
- May post event when accept mutex is active (`ngx_post_event(rev, &ngx_posted_events)`)
- Arms preread/proxy timers and read interest
- Resumes through `ngx_stream_session_handler()` → `ngx_stream_core_run_phases()`

### 2.2 Integration with `src/core`

Stream relies on core for:

- Module and config framework (`ngx_module_t`, `ngx_command_t`)
- Memory pools and connection abstractions
- Logging and socket/address primitives
- Hash tables for variable and server name lookup

Session allocations are connection-pool lifetime scoped.

### 2.3 Integration with stream submodules

Phase handlers are provided by stream modules:

- **Post-accept**: `ngx_stream_realip_module`, `ngx_stream_limit_conn_module`
- **Access**: `ngx_stream_access_module`
- **SSL**: `ngx_stream_ssl_module`
- **Preread**: `ngx_stream_ssl_preread_module`
- **Content**: `ngx_stream_proxy_module`, `ngx_stream_return_module`
- **Log**: `ngx_stream_log_module`

Additional modules: `ngx_stream_map_module`, `ngx_stream_geo_module`, `ngx_stream_upstream_*` modules.

The core module assembles handlers from all modules into `phase_engine` at postconfiguration.

### 2.4 Integration with upstream and networking

Upstream modules provide:

- Load balancing: round-robin, hash, least-conn, random
- Peer selection and connection management
- Health checking integration

The proxy module (`ngx_stream_proxy_module`) is typically the content handler, managing bidirectional data relay between client and upstream.

---

## 3. Basic Code Walkthrough for Stream Flows

### 3.1 Generic tracing template

1. Accept path calls `ngx_stream_init_connection()`
2. Resolve server context for local `address:port`
3. Allocate session and module context/variables
4. Run PROXY protocol preread if enabled
5. Enter `ngx_stream_session_handler()` → `ngx_stream_core_run_phases()`
6. Phase handlers advance, defer, or finalize session

### 3.2 Concrete walkthrough: connection initialization

#### Step A: address resolution (`ngx_stream_init_connection()`, handler.c line 20)

1. `port = c->listening->servers` — get `ngx_stream_port_t`
2. If `port->naddrs > 1`: call `ngx_connection_local_sockaddr()`, loop through addresses to find matching `ngx_stream_addr_conf_t`
3. If single address: use `addr[0].conf` directly

#### Step B: session allocation

1. `s = ngx_pcalloc(c->pool, sizeof(ngx_stream_session_t))`
2. `s->signature = NGX_STREAM_MODULE` — `"STRM"` magic
3. `s->main_conf = ctx->main_conf`; `s->srv_conf = ctx->srv_conf`
4. `s->virtual_names = addr_conf->virtual_names`
5. `s->connection = c`; `c->data = s`
6. If connection has buffered data from accept: `s->received += c->buffer->last - c->buffer->pos`

#### Step C: context and variable setup

1. `s->ctx = ngx_pcalloc(c->pool, sizeof(void *) * ngx_stream_max_module)` — per-module context array
2. `s->variables = ngx_pcalloc(...)` — pre-allocate variable value slots
3. Record `s->start_sec`, `s->start_msec` from cached time
4. Set `rev->handler = ngx_stream_session_handler`

#### Step D: PROXY protocol branch

If `addr_conf->proxy_protocol`:
1. Set `rev->handler = ngx_stream_proxy_protocol_handler`
2. If data not ready: arm timer (`cscf->proxy_protocol_timeout`), register read event, return
3. When data arrives: `recv(MSG_PEEK)` → `ngx_proxy_protocol_read()` → consume header
4. On success: call `ngx_stream_session_handler(rev)` to enter phase engine

#### Step E: accept mutex deferral

If `ngx_use_accept_mutex`: post read event to deferred queue, return. Event will fire on next cycle iteration.

### 3.3 Concrete walkthrough: phase engine execution

#### `ngx_stream_core_run_phases()` (core_module.c line 169)

```c
ph = cmcf->phase_engine.handlers;
while (ph[s->phase_handler].checker) {
    rc = ph[s->phase_handler].checker(s, &ph[s->phase_handler]);
    if (rc == NGX_OK) {
        return;  // handler took control or finalized session
    }
    // rc == NGX_AGAIN → continue to next handler
}
```

The loop walks the flat `phase_engine.handlers[]` array. Each handler slot has:
- A `checker` appropriate to its phase (generic, preread, or content)
- The `handler` function from the module
- A `next` index pointing to the first handler of the next phase

#### Generic phase checker (`ngx_stream_core_generic_phase`, core_module.c line 190)

Used by POST_ACCEPT, PREACCESS, ACCESS, SSL phases:

| Handler returns | Checker action |
|----------------|----------------|
| `NGX_OK` | `s->phase_handler = ph->next` (skip to next phase), return `NGX_AGAIN` |
| `NGX_DECLINED` | `s->phase_handler++` (try next handler), return `NGX_AGAIN` |
| `NGX_AGAIN` / `NGX_DONE` | return `NGX_OK` (handler will resume later) |
| `NGX_ERROR` | finalize session, return `NGX_OK` |

#### Preread phase checker (`ngx_stream_core_preread_phase`, core_module.c line 232)

Additional logic beyond generic checker:

1. Check for read timeout → finalize
2. If first call and handler returns `NGX_AGAIN`: allocate `c->buffer` (`preread_buffer_size`)
3. Use `MSG_PEEK` or regular read to fill preread buffer
4. On `NGX_AGAIN`: arm read event + timer, set `c->read->handler = ngx_stream_session_handler`, return `NGX_OK`
5. On completion: delete timer, advance phase

#### Content phase checker (`ngx_stream_core_content_phase`, core_module.c line 441)

1. Set `c->log->action = NULL`
2. Enable `TCP_NODELAY` if configured
3. Call `cscf->handler(s)` — the content handler (usually `ngx_stream_proxy_handler`)
4. Return `NGX_OK` unconditionally — content handler takes full control

### 3.4 Concrete walkthrough: session finalization

`ngx_stream_finalize_session()` (handler.c line 301):

1. `s->status = rc` — store final status code
2. `ngx_stream_log_session(s)` — invoke all `NGX_STREAM_LOG_PHASE` handlers directly
3. `ngx_stream_close_connection(c)` →
   - If SSL: `ngx_ssl_shutdown()` (may defer close asynchronously)
   - Decrement `ngx_stat_active`
   - Save `pool = c->pool`
   - `ngx_close_connection(c)` — remove events, close fd, return slot
   - `ngx_destroy_pool(pool)` — free all session memory

---

## 4. Fast Mental Model for New Engineers

- **Session = connection**: `ngx_stream_session_t` wraps `ngx_connection_t` for L4 processing
- **Phase engine = flat handler array**: compiled at config time, walked sequentially at runtime
- **Checkers interpret handler results**: different phases have different checker semantics
- **Content handler takes full control**: once content phase fires, the handler owns the session
- **Preread = peek before deciding**: modules like `ssl_preread` inspect initial bytes without consuming them
- **Log phase is special**: not part of `run_phases()` — invoked directly by `finalize_session()`
- **Everything dies with the pool**: `ngx_stream_close_connection()` destroys connection pool

---

## 5. Recommended First Read Order in `src/stream`

1. `src/stream/ngx_stream.h` — all struct definitions, phase enum, module types
2. `src/stream/ngx_stream_handler.c` — session bootstrap, PROXY protocol, finalize/close
3. `src/stream/ngx_stream_core_module.c` — phase engine (`run_phases`), checkers, core directives
4. `src/stream/ngx_stream.c` — config block parsing, phase/listener graph assembly
5. `src/stream/ngx_stream_proxy_module.c` — primary content handler, upstream proxy
6. `src/stream/ngx_stream_upstream.c` — upstream framework
7. `src/stream/ngx_stream_variables.c` — variable framework
8. `src/stream/ngx_stream_ssl_preread_module.c` — example preread phase handler

---

## 6. Practical Debug Checklist

When debugging stream module behavior:

1. Did `addr_conf` mapping pick the intended stream server block? Check address matching in `ngx_stream_init_connection()`.
2. Is PROXY protocol branch active? Check `addr_conf->proxy_protocol` and `rev->handler` after accept.
3. Is the PROXY protocol timeout firing? Check timer registration and `cscf->proxy_protocol_timeout` value.
4. What is `s->phase_handler`? Map the index to `phase_engine.handlers[]` to identify which phase/handler is executing.
5. What did the handler return? Trace through the appropriate checker to understand advancement behavior.
6. Is the preread buffer allocated and filling? Check `c->buffer` and `preread_buffer_size`.
7. Are preread timers added and removed on all branches? Missing `ngx_del_timer()` causes timeout after preread completes.
8. Is `cscf->handler` set for the content phase? `NULL` handler causes `INTERNAL_SERVER_ERROR`.
9. Is the content handler returning to event loop correctly? Content handler must manage its own read/write events.
10. Did finalize path log the session? Check `NGX_STREAM_LOG_PHASE` handlers are registered.
11. Is SSL shutdown completing? `ngx_ssl_shutdown()` returning `NGX_AGAIN` defers close — check for stalled connections.
