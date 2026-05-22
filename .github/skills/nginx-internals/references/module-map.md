# Module Map — nginx Source Tree

> **Source paths**: All `src/` paths below are relative to the **nginx repository root** (i.e. `<repo>/src/`), not relative to `.github/`.

Quick lookup: which module owns what behavior, key files per module, and cross-module dependencies.

## src/core — Process Runtime Foundation

**Owns**: Process startup, cycle lifecycle, config parsing, connection objects, memory pools, containers, logging, time cache

| File | Purpose |
|------|---------|
| `nginx.c` | `main()` — CLI parsing, bootstrap sequence, process cycle entry |
| `ngx_cycle.c` | `ngx_init_cycle()` — config reload, cycle construction |
| `ngx_connection.c` | Connection free-list management, listening socket setup |
| `ngx_palloc.c` | Pool allocator (small/large/block paths) |
| `ngx_conf_file.c` | Config tokenizer and directive dispatcher |
| `ngx_module.c` | Module index assignment and lifecycle |
| `ngx_log.c` | Logging subsystem |
| `ngx_resolver.c` | Async DNS resolver |
| `ngx_buf.c` | Buffer and chain utilities |
| `ngx_hash.c` | Hash table (used for headers, server names) |
| `ngx_rbtree.c` | Red-black tree (used for timers, caches) |
| `ngx_shmtx.c` | Shared memory mutex (accept mutex, slab locks) |
| `ngx_slab.c` | Slab allocator for shared memory zones |

**Depends on**: `src/os` (pagesize, cacheline, rlimits)
**Depended on by**: Everything

## src/event — I/O Readiness and Timer Management

**Owns**: Event loop, backend multiplexing (epoll/kqueue/select), accept synchronization, timer rbtree, posted event queues

| File | Purpose |
|------|---------|
| `ngx_event.c` | `ngx_process_events_and_timers()` — main loop, worker init |
| `ngx_event_accept.c` | `ngx_event_accept()` — connection accept path |
| `ngx_event_timer.c` | Timer rbtree management |
| `ngx_event_posted.c` | Posted/deferred event queue processing |
| `ngx_event_connect.c` | Outbound connection establishment |
| `ngx_event_pipe.c` | Upstream pipe buffering |
| `modules/ngx_epoll_module.c` | Linux epoll backend |
| `modules/ngx_kqueue_module.c` | BSD/macOS kqueue backend |
| `modules/ngx_select_module.c` | Portable select fallback |

**Depends on**: `src/core` (connections, pools, logging), `src/os` (I/O vtable)
**Depended on by**: `src/http`, `src/stream`, `src/mail`

## src/http — HTTP Protocol Processing

**Owns**: HTTP/1.1, HTTP/2, HTTP/3(QUIC) protocol handling, 11-phase request pipeline, header/body filter chains, upstream proxy, caching, SSL integration

| File | Purpose |
|------|---------|
| `ngx_http.c` | `http {}` block parser, phase/listener graph assembly |
| `ngx_http_core_module.c` | Phase engine, core directives, location matching |
| `ngx_http_request.c` | `ngx_http_init_connection()`, request lifecycle |
| `ngx_http_parse.c` | HTTP request line and header parsers (state machines) |
| `ngx_http_upstream.c` | Upstream proxy framework |
| `ngx_http_variables.c` | Variable evaluation framework |
| `ngx_http_script.c` | Rewrite scripting engine |
| `ngx_http_file_cache.c` | Proxy/fastcgi cache |
| `v2/ngx_http_v2.c` | HTTP/2 connection and stream management |
| `v3/ngx_http_v3.c` | HTTP/3 QUIC integration |

**11 Phases**: POST_READ → SERVER_REWRITE → FIND_CONFIG → REWRITE → POST_REWRITE → PREACCESS → ACCESS → POST_ACCESS → PRECONTENT → CONTENT → LOG

**Depends on**: `src/core`, `src/event`, `src/os`

## src/stream — Layer-4 TCP/UDP Proxy

**Owns**: `stream {}` config, 7-phase pipeline, TCP/UDP session lifecycle, upstream proxy, SSL preread

| File | Purpose |
|------|---------|
| `ngx_stream.c` | `stream {}` block parser, phase graph assembly |
| `ngx_stream_handler.c` | Session bootstrap, PROXY protocol, finalize/close |
| `ngx_stream_core_module.c` | Phase engine, checkers, core directives |
| `ngx_stream_proxy_module.c` | Content handler — upstream TCP/UDP proxy |
| `ngx_stream_upstream.c` | Upstream framework |
| `ngx_stream_ssl_preread_module.c` | SNI/ALPN detection without terminating SSL |

**7 Phases**: POST_ACCEPT → PREACCESS → ACCESS → SSL → PREREAD → CONTENT → LOG

**Depends on**: `src/core`, `src/event`

## src/mail — Mail Protocol Frontend

**Owns**: POP3/IMAP/SMTP frontends, auth HTTP proxy, upstream mail relay

| File | Purpose |
|------|---------|
| `ngx_mail.c` | `mail {}` block parser |
| `ngx_mail_handler.c` | Session bootstrap, init, auth dispatch, close |
| `ngx_mail_core_module.c` | Core directives and server config |
| `ngx_mail_smtp_handler.c` | SMTP state machine (16 states) |
| `ngx_mail_imap_handler.c` | IMAP state machine (9 states) |
| `ngx_mail_pop3_handler.c` | POP3 state machine (8 states) |
| `ngx_mail_auth_http_module.c` | External auth via HTTP subrequest |
| `ngx_mail_proxy_module.c` | Upstream mail relay |

**Depends on**: `src/core`, `src/event`

## src/os — Platform Adaptation Layer

**Owns**: OS-specific syscalls, process control, signal handling, I/O function vtable, platform init

| File | Purpose |
|------|---------|
| `unix/ngx_posix_init.c` | `ngx_os_init()` — pagesize, cacheline, ncpu, rlimits |
| `unix/ngx_process.c` | `ngx_spawn_process()`, `ngx_init_signals()` |
| `unix/ngx_process_cycle.c` | Master/worker loops, signal dispatch |
| `unix/ngx_channel.c` | Master↔worker IPC (Unix socketpairs) |
| `unix/ngx_daemon.c` | Double-fork daemonization |
| `unix/ngx_alloc.c` | `malloc`/`posix_memalign` wrappers |
| `unix/ngx_linux_init.c` | Linux-specific init (sendfile flags) |
| `unix/ngx_linux_sendfile_chain.c` | Linux sendfile + writev |

**Depends on**: libc/POSIX
**Depended on by**: `src/core`, `src/event`

## src/misc — Special-Purpose Modules

| File | Purpose |
|------|---------|
| `ngx_google_perftools_module.c` | Per-worker CPU profiling via Google perftools |
| `ngx_cpp_test_module.cpp` | C++ header compatibility test (build-only) |

## Cross-Module Dependency Graph

```
src/os  ──►  src/core  ──►  src/event  ──►  src/http
                │                │           src/stream
                │                │           src/mail
                │                │
                └── src/misc ────┘
```

All protocol modules (http/stream/mail) depend on both core and event. OS is the foundation layer.
