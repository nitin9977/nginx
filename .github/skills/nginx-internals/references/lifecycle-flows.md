# Lifecycle Flows — nginx Execution Paths

Step-by-step execution paths for key operations. Use to trace behavior without reading source.

## 1. Process Startup (main → workers)

```
main() [src/core/nginx.c:197]
  ├─ ngx_get_options()              — parse CLI args
  ├─ ngx_time_init()                — cache current time
  ├─ ngx_log_init()                 — open error log
  ├─ ngx_ssl_init()                 — OpenSSL global init
  ├─ create init_cycle with pool
  ├─ ngx_os_init()                  — pagesize, ncpu, rlimits
  ├─ ngx_crc32_table_init()         — needs cacheline from os_init
  ├─ ngx_add_inherited_sockets()    — hot-upgrade fd inheritance
  ├─ ngx_preinit_modules()          — assign module indices
  ├─ ngx_init_cycle(&init_cycle)    — FULL config parse + setup
  ├─ ngx_init_signals()             — install signal handlers
  ├─ ngx_daemon()                   — daemonize if configured
  ├─ ngx_create_pidfile()           — write PID
  └─ ngx_master_process_cycle()     — enter master loop
       ├─ ngx_start_worker_processes()  — fork workers
       └─ for(;;) sigsuspend() → dispatch signals
```

## 2. Config Reload (SIGHUP)

```
Master receives SIGHUP → ngx_reconfigure = 1
  │
  ├─ ngx_init_cycle(old_cycle)      — build NEW cycle
  │    ├─ new pool, new conf_ctx
  │    ├─ create_conf for all core modules
  │    ├─ ngx_conf_parse()          — parse config into new structs
  │    ├─ init_conf for all core modules
  │    ├─ open log files, shared memory, listeners
  │    └─ init_module for all modules
  │
  ├─ If fails: continue with old cycle, log error
  │
  ├─ ngx_start_worker_processes(new_cycle, JUST_RESPAWN)
  ├─ ngx_start_cache_manager_processes(new_cycle)
  ├─ ngx_msleep(100)               — let new workers init
  └─ ngx_signal_worker_processes(old, SHUTDOWN_SIGNAL)
       └─ old workers drain connections then exit
```

## 3. Worker Event Loop

```
ngx_worker_process_cycle() [src/os/unix/ngx_process_cycle.c]
  ├─ ngx_worker_process_init()
  │    ├─ set CPU affinity, rlimits, priority
  │    ├─ setgid/setuid (privilege drop)
  │    ├─ call init_process on all modules
  │    └─ register channel read event
  │
  └─ for(;;)
       ├─ ngx_process_events_and_timers(cycle)
       │    ├─ [optional] try accept mutex
       │    ├─ ngx_process_events()       — epoll_wait/kevent/select
       │    ├─ ngx_event_expire_timers()  — fire expired timers
       │    ├─ ngx_event_process_posted() — process accept queue
       │    └─ ngx_event_process_posted() — process data queue
       │
       ├─ if ngx_terminate → exit immediately
       ├─ if ngx_quit → close listeners, set exiting, drain
       └─ if ngx_reopen → reopen log files
```

## 4. Connection Accept Path

```
epoll_wait returns EPOLLIN on listener fd
  │
  ├─ ngx_event_accept() [src/event/ngx_event_accept.c]
  │    ├─ accept4()/accept() → raw fd
  │    ├─ ngx_get_connection(fd)
  │    │    ├─ pop from cycle->free_connections
  │    │    ├─ zero connection and event structs
  │    │    └─ toggle rev->instance bit (stale-event guard)
  │    ├─ create c->pool (ls->pool_size)
  │    ├─ set c->recv/send from ngx_os_io
  │    ├─ set c->local_sockaddr, c->sockaddr
  │    └─ ls->handler(c) → protocol takes ownership
  │
  └─ Protocol handlers:
       ├─ HTTP:   ngx_http_init_connection(c)
       ├─ Stream: ngx_stream_init_connection(c)
       └─ Mail:   ngx_mail_init_connection(c)
```

## 5. Connection Close Path

```
Protocol calls ngx_close_connection(c)
  ├─ ngx_del_timer(c->read) if timer_set
  ├─ ngx_del_timer(c->write) if timer_set
  ├─ ngx_del_conn(c) or ngx_del_event(read) + ngx_del_event(write)
  ├─ remove from posted event queues
  ├─ c->read->closed = 1; c->write->closed = 1
  ├─ ngx_reusable_connection(c, 0)  — remove from reusable queue
  ├─ ngx_free_connection(c)         — push back to free list
  └─ close(c->fd)
```

## 6. Pool Allocator Flow

```
ngx_palloc(pool, size)
  │
  ├─ size ≤ pool->max (SMALL PATH)
  │    └─ ngx_palloc_small(pool, size, align)
  │         ├─ walk from pool->current
  │         ├─ if d.last + size ≤ d.end → bump d.last, return
  │         └─ if exhausted → ngx_palloc_block()
  │              ├─ alloc new block (pool size)
  │              ├─ chain via d.next
  │              ├─ if old block d.failed > 4 → advance current
  │              └─ bump d.last in new block, return
  │
  └─ size > pool->max (LARGE PATH)
       └─ ngx_palloc_large(pool, size)
            ├─ ngx_alloc(size) → malloc
            └─ link into pool->large list
```

## 7. HTTP Request Lifecycle

```
ngx_http_init_connection(c)
  ├─ allocate ngx_http_connection_t
  ├─ set rev->handler = ngx_http_wait_request_handler
  ├─ arm read timer (client_header_timeout)
  │
  └─ [data arrives]
       ├─ ngx_http_wait_request_handler(rev)
       │    ├─ allocate request buffer
       │    ├─ recv() data
       │    └─ ngx_http_create_request(c) → ngx_http_request_t
       │
       ├─ ngx_http_process_request_line(rev)
       │    └─ ngx_http_parse_request_line() — state machine (19+ states)
       │
       ├─ ngx_http_process_request_headers(rev)
       │    └─ ngx_http_parse_header_line() per header
       │
       ├─ ngx_http_process_request(r)
       │    └─ ngx_http_handler(r)
       │         └─ ngx_http_core_run_phases(r)
       │              └─ walk phase_engine.handlers[]
       │
       └─ Response:
            ├─ ngx_http_send_header(r) → header filter chain
            ├─ ngx_http_output_filter(r, chain) → body filter chain
            └─ ngx_http_finalize_request(r, rc)
```

## 8. Stream Session Lifecycle

```
ngx_stream_init_connection(c)
  ├─ resolve addr_conf from local address:port
  ├─ allocate ngx_stream_session_t
  ├─ set signature="STRM", attach main_conf/srv_conf
  ├─ allocate ctx[] and variables[]
  ├─ set rev->handler = ngx_stream_session_handler
  │
  ├─ [if proxy_protocol] → ngx_stream_proxy_protocol_handler
  │    ├─ recv(MSG_PEEK) → ngx_proxy_protocol_read()
  │    └─ on success → ngx_stream_session_handler(rev)
  │
  └─ ngx_stream_session_handler(rev)
       └─ ngx_stream_core_run_phases(s)
            └─ while (ph[s->phase_handler].checker)
                 ├─ checker returns NGX_OK → stop (handler controls)
                 └─ checker returns NGX_AGAIN → next handler
```

## 9. Mail Session Lifecycle

```
ngx_mail_init_connection(c)
  ├─ resolve addr_conf from local address:port
  ├─ allocate ngx_mail_session_t
  ├─ set signature="MAIL", attach main_conf/srv_conf
  │
  ├─ [if SSL] → ngx_mail_ssl_init_connection()
  ├─ [if proxy_protocol] → ngx_mail_proxy_protocol_handler
  │
  └─ ngx_mail_init_session(c)
       ├─ set s->protocol from cscf->protocol->type
       ├─ allocate ctx[] array
       ├─ set c->write->handler = ngx_mail_send
       └─ cscf->protocol->init_session(s, c) → send greeting

Auth flow:
  protocol handler parses credentials
    → ngx_mail_auth(s, c)
      ├─ reset parser state, increment login_attempt
      └─ ngx_mail_auth_http_init(s) → HTTP subrequest to auth backend
           ├─ success → ngx_mail_proxy_init() → connect upstream
           └─ failure → send error, check max_errors
```

## 10. Master Signal Dispatch

```
Master loop: for(;;) { sigsuspend(&set); ... }

Signal flag → Action:
  ngx_reap (SIGCHLD)       → ngx_reap_children() — waitpid, respawn
  ngx_terminate (SIGTERM)   → TERM to workers, escalate to SIGKILL
  ngx_quit (SIGQUIT)        → QUIT to workers, close listeners
  ngx_reconfigure (SIGHUP)  → ngx_init_cycle(), new workers, shutdown old
  ngx_reopen (SIGUSR1)      → reopen files, signal workers
  ngx_change_binary (USR2)  → ngx_exec_new_binary() — hot upgrade
  ngx_noaccept (SIGWINCH)   → stop accepting, shutdown workers

Termination escalation:
  delay starts at 50ms, doubles each SIGALRM
  if delay > 1000ms → SIGKILL to workers
```
