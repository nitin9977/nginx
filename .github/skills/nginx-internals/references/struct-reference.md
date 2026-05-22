# Struct Reference — nginx Core Data Structures

> **Source paths**: All `src/` paths below are relative to the **nginx repository root** (i.e. `<repo>/src/`), not relative to `.github/`.

Field-level descriptions of critical structs so you don't need to read headers on every prompt.

## ngx_cycle_t (`src/core/ngx_cycle.h`)

Global runtime snapshot — one per active configuration.

| Field | Type | Purpose |
|-------|------|---------|
| `conf_ctx` | `void ****` | Module config tree: `conf_ctx[module->index]` |
| `pool` | `ngx_pool_t *` | Cycle-lifetime memory pool |
| `modules` / `modules_n` | `ngx_module_t **` / `ngx_uint_t` | Ordered module array |
| `connections` | `ngx_connection_t *` | Pre-allocated connection slot array |
| `read_events` / `write_events` | `ngx_event_t *` | Parallel event arrays (indexed same as connections) |
| `free_connections` / `free_connection_n` | `ngx_connection_t *` / `ngx_uint_t` | Free-list head and count |
| `listening` | `ngx_array_t` | Array of `ngx_listening_t` bound sockets |
| `shared_memory` | `ngx_list_t` | `ngx_shm_zone_t` zones |
| `old_cycle` | `ngx_cycle_t *` | Previous cycle (during reload) |

## ngx_connection_t (`src/core/ngx_connection.h`)

Transport endpoint state per socket.

| Field | Type | Purpose |
|-------|------|---------|
| `data` | `void *` | Protocol session (HTTP request, mail session, etc.) |
| `read` / `write` | `ngx_event_t *` | Per-direction event state |
| `fd` | `ngx_socket_t` | Socket descriptor |
| `recv` / `send` | fn ptrs | I/O functions (platform-specific) |
| `recv_chain` / `send_chain` | fn ptrs | Chain I/O (readv/writev/sendfile) |
| `listening` | `ngx_listening_t *` | Back-pointer to listener |
| `pool` | `ngx_pool_t *` | Per-connection memory pool |
| `number` | `ngx_atomic_uint_t` | Monotonic connection counter |
| `idle` / `reusable` / `close` | `unsigned:1` | Keepalive lifecycle flags |
| `destroyed` | `unsigned:1` | Set during close to prevent double-free |

## ngx_event_t (`src/event/ngx_event.h`)

Per-direction (read or write) event state.

| Field | Type | Purpose |
|-------|------|---------|
| `data` | `void *` | Back-pointer to owning `ngx_connection_t` |
| `handler` | `ngx_event_handler_pt` | Current callback — THIS IS THE STATE |
| `ready` | `unsigned:1` | Kernel says I/O is possible |
| `active` | `unsigned:1` | Registered with event backend |
| `timer_set` | `unsigned:1` | Timer armed in rbtree |
| `timedout` | `unsigned:1` | Timer expired |
| `posted` | `unsigned:1` | In posted event queue |
| `instance` | `unsigned:1` | Stale-event guard bit (toggled on reuse) |
| `write` | `unsigned:1` | 1=write event, 0=read event |
| `eof` | `unsigned:1` | Peer closed connection |
| `error` | `unsigned:1` | Socket error detected |
| `available` | `int` | Bytes available (kqueue) or accept hint |

## ngx_pool_t (`src/core/ngx_palloc.h`)

Region allocator for bounded-lifetime data.

| Field | Type | Purpose |
|-------|------|---------|
| `d.last` | `u_char *` | Bump pointer — next allocation starts here |
| `d.end` | `u_char *` | End of current block |
| `d.next` | `ngx_pool_t *` | Next block in chain |
| `d.failed` | `ngx_uint_t` | Allocation failures on this block (triggers `current` advance after 4) |
| `max` | `size_t` | Small vs large threshold |
| `current` | `ngx_pool_t *` | First block still worth trying |
| `large` | `ngx_pool_large_t *` | Linked list of `malloc`'d allocations |
| `cleanup` | `ngx_pool_cleanup_t *` | Cleanup callbacks run on `ngx_destroy_pool()` |

## ngx_event_actions_t (`src/event/ngx_event.h`)

Backend vtable — one implementation per OS (epoll, kqueue, select).

| Field | Signature | Purpose |
|-------|-----------|---------|
| `add` | `(ev, event, flags)` | Register event with kernel |
| `del` | `(ev, event, flags)` | Deregister event |
| `enable` / `disable` | `(ev, event, flags)` | Enable/disable without deregister |
| `add_conn` / `del_conn` | `(c, flags)` | Register/deregister entire connection |
| `notify` | `(handler)` | Wake up event loop (eventfd/pipe) |
| `process_events` | `(cycle, timer, flags)` | Poll kernel for ready events |
| `init` / `done` | `(cycle, timer)` | Backend init/cleanup |

## ngx_http_request_t (`src/http/ngx_http_request.h`)

Per-HTTP-request state (abbreviated — full struct is ~200 fields).

| Field | Type | Purpose |
|-------|------|---------|
| `connection` | `ngx_connection_t *` | Transport connection |
| `pool` | `ngx_pool_t *` | Request-lifetime pool |
| `header_in` | `ngx_buf_t *` | Input buffer for headers |
| `headers_in` / `headers_out` | structs | Parsed request/response headers |
| `ctx` | `void **` | Per-module context array |
| `main_conf` / `srv_conf` / `loc_conf` | `void **` | Config context arrays |
| `upstream` | `ngx_http_upstream_t *` | Upstream proxy state |
| `phase_handler` | `ngx_int_t` | Current phase engine index |
| `uri` / `args` / `method` | various | Parsed request line fields |
| `main` / `parent` | `ngx_http_request_t *` | Main request and subrequest parent |
| `count` | `ngx_uint_t` | Reference count (subrequests + 1) |

## ngx_stream_session_t (`src/stream/ngx_stream.h`)

Per-stream-connection session state.

| Field | Type | Purpose |
|-------|------|---------|
| `signature` | `uint32_t` | Magic `"STRM"` (0x4d525453) |
| `connection` | `ngx_connection_t *` | Transport connection |
| `received` | `off_t` | Total bytes from client |
| `start_sec` / `start_msec` | time types | Session start timestamp |
| `ctx` / `main_conf` / `srv_conf` | `void **` | Module config/context arrays |
| `upstream` | `ngx_stream_upstream_t *` | Upstream proxy state |
| `variables` | `ngx_stream_variable_value_t *` | Pre-allocated variable array |
| `phase_handler` | `ngx_int_t` | Current phase engine index |
| `status` | `ngx_uint_t` | Final status (200/400/403/500/502/503) |
| `ssl:1` / `health_check:1` | bit fields | Connection type flags |

## ngx_mail_session_t (`src/mail/ngx_mail.h`)

Per-mail-connection session state.

| Field | Type | Purpose |
|-------|------|---------|
| `signature` | `uint32_t` | Magic `"MAIL"` (0x4C49414D) |
| `connection` | `ngx_connection_t *` | Transport connection |
| `buffer` | `ngx_buf_t *` | Input buffer for command parsing |
| `ctx` / `main_conf` / `srv_conf` | `void **` | Module config/context arrays |
| `proxy` | `ngx_mail_proxy_ctx_t *` | Upstream relay state |
| `mail_state` | `ngx_uint_t` | Protocol-independent state index |
| `protocol:3` | bit field | POP3=0, IMAP=1, SMTP=2 |
| `quit:1` / `blocked:1` / `starttls:1` | bit fields | Session state flags |
| `auth_method:3` | bit field | Current auth mechanism |
| `login` / `passwd` | `ngx_str_t` | Extracted credentials |
| `smtp_helo` / `smtp_from` / `smtp_to` | `ngx_str_t` | SMTP envelope state |
| `errors` / `login_attempt` | `ngx_uint_t` | Failure counters |

## ngx_process_t (`src/os/unix/ngx_process.h`)

Per-process state in `ngx_processes[NGX_MAX_PROCESSES]`.

| Field | Type | Purpose |
|-------|------|---------|
| `pid` | `ngx_pid_t` | Process ID |
| `status` | `int` | `waitpid()` exit status |
| `channel[2]` | `ngx_socket_t` | Socketpair for master↔worker IPC |
| `proc` | `ngx_spawn_proc_pt` | Worker main function callback |
| `data` | `void *` | Opaque data for `proc` |
| `name` | `char *` | Process name string |
| `respawn:1` / `exiting:1` / `exited:1` | bit fields | Process lifecycle flags |

## ngx_os_io_t (`src/os/unix/ngx_os.h`)

I/O function vtable wired into connections at accept.

| Field | Implementation | Purpose |
|-------|---------------|---------|
| `recv` | `ngx_unix_recv` | Single-buffer `recv()` |
| `recv_chain` | `ngx_readv_chain` | Scatter/gather `readv()` |
| `udp_recv` | `ngx_udp_unix_recv` | UDP `recvmsg()` |
| `send` | `ngx_unix_send` | Single-buffer `send()` |
| `udp_send` | `ngx_udp_unix_send` | UDP `sendto()` |
| `send_chain` | `ngx_writev_chain` | Gather write (replaced by sendfile per-platform) |

## ngx_stream_phase_handler_t (`src/stream/ngx_stream.h`)

Phase engine slot.

| Field | Type | Purpose |
|-------|------|---------|
| `checker` | `ngx_stream_phase_handler_pt` | Phase-specific checker (generic/preread/content) |
| `handler` | `ngx_stream_handler_pt` | Module's handler function |
| `next` | `ngx_uint_t` | Index of first handler in next phase |

## ngx_mail_protocol_t (`src/mail/ngx_mail.h`)

Protocol vtable — one per protocol (POP3, IMAP, SMTP).

| Field | Type | Purpose |
|-------|------|---------|
| `name` | `ngx_str_t` | Protocol name |
| `type` | `ngx_uint_t` | Protocol type enum |
| `init_session` | fn ptr | Send initial greeting |
| `init_protocol` | fn ptr | Set read handler for command loop |
| `parse_command` | fn ptr | Command parser automaton |
| `auth_state` | fn ptr | Auth state machine handler |
| `internal_server_error` / `cert_error` / `no_cert` | `ngx_str_t` | Canned error responses |
