---
name: nginx-internals
description: 'nginx codebase internals knowledge base. USE FOR: debugging nginx issues (crashes, memory leaks, connection problems, config reload failures, signal handling, event loop stalls, phase engine bugs, upstream failures, protocol parsing errors), designing new features or modules, tracing request/connection lifecycle, understanding struct field semantics, identifying which source files own specific behavior. DO NOT USE FOR: nginx configuration syntax help, general C programming questions. INVOKES: read_file for source verification.'
argument-hint: 'Describe the issue to debug or feature to design'
---

# nginx Internals Knowledge Base

> **Source paths**: All `src/` references in this file and its references are relative to the **nginx repository root** (i.e. `<repo>/src/`), not relative to `.github/`.

Distilled from 14 architecture documents covering all `src/` modules. Use this skill to avoid full codebase read-throughs on every prompt.

## When to Use

- Debugging crashes, segfaults, or undefined behavior in nginx
- Tracing memory leaks or use-after-free issues
- Understanding connection lifecycle or event loop behavior
- Designing new modules or features
- Investigating config reload failures or signal handling issues
- Analyzing phase engine execution (HTTP or stream)
- Debugging upstream proxy or load balancing problems
- Understanding buffer management and parsing state machines

## Procedure

1. **Identify the subsystem**: Use the [Module Map](./references/module-map.md) to locate which module owns the behavior
2. **Load struct knowledge**: Use the [Struct Reference](./references/struct-reference.md) for field-level understanding without reading headers
3. **Trace the flow**: Use the [Lifecycle Flows](./references/lifecycle-flows.md) for step-by-step execution paths
4. **Check known failure modes**: Use the [Debug Playbook](./references/debug-playbook.md) for known issue patterns and root cause checklists
5. **Verify in source**: Use `read_file` on the specific source file + line range identified by the references
6. **For new features**: Use the [Design Patterns](./references/design-patterns.md) for module registration, phase handler, and config directive patterns
7. **For writing/reviewing code**: Use the [nginx Coding Style Guide](../../docs/coding-style/NGINX_CODING_STYLE.md) — mandatory formatting, naming, and structural rules from the official nginx development guide

## Key Invariants (Always True)

- Workers are isolated processes — no in-process mutexes protect connection/pool state
- `ngx_cycle_t` is THE global context — all runtime state hangs off it
- Config reload = new cycle creation, not in-place mutation
- Connections are pre-allocated slots reused via free list with `instance` bit toggle
- Pools are scoped lifetimes: cycle > connection > request
- Signal handlers only set `sig_atomic_t` flags; master loop dispatches
- Phase engines are flat arrays walked sequentially; checkers interpret handler return codes
- Content handler takes full control — once it fires, it owns the session/request
- Everything dies with its pool — `ngx_destroy_pool()` is the universal cleanup

## Quick Lookup: "Where Does X Live?"

| Behavior | Owner Module | Entry Point |
|----------|-------------|-------------|
| Process startup | `src/core/nginx.c` | `main()` |
| Config parse | `src/core/ngx_conf_file.c` | `ngx_conf_parse()` |
| Cycle lifecycle | `src/core/ngx_cycle.c` | `ngx_init_cycle()` |
| Connection alloc/free | `src/core/ngx_connection.c` | `ngx_get_connection()` / `ngx_close_connection()` |
| Pool allocator | `src/core/ngx_palloc.c` | `ngx_palloc()` / `ngx_destroy_pool()` |
| Event loop | `src/event/ngx_event.c` | `ngx_process_events_and_timers()` |
| epoll backend | `src/event/modules/ngx_epoll_module.c` | `ngx_epoll_process_events()` |
| Accept path | `src/event/ngx_event_accept.c` | `ngx_event_accept()` |
| Timers | `src/event/ngx_event_timer.c` | `ngx_event_expire_timers()` |
| HTTP phases | `src/http/ngx_http_core_module.c` | `ngx_http_core_run_phases()` |
| HTTP parsing | `src/http/ngx_http_parse.c` | `ngx_http_parse_request_line()` |
| HTTP connection init | `src/http/ngx_http_request.c` | `ngx_http_init_connection()` |
| HTTP upstream | `src/http/ngx_http_upstream.c` | `ngx_http_upstream_init()` |
| Stream phases | `src/stream/ngx_stream_core_module.c` | `ngx_stream_core_run_phases()` |
| Stream session | `src/stream/ngx_stream_handler.c` | `ngx_stream_init_connection()` |
| Mail session | `src/mail/ngx_mail_handler.c` | `ngx_mail_init_connection()` |
| Master/worker loops | `src/os/unix/ngx_process_cycle.c` | `ngx_master_process_cycle()` |
| OS init | `src/os/unix/ngx_posix_init.c` | `ngx_os_init()` |
| Signal handling | `src/os/unix/ngx_process.c` | `ngx_init_signals()` |

## Directive Reference (Configuration)

All nginx configuration directives documented from https://nginx.org/en/docs/.
Use these when you need to know exact directive syntax, defaults, or valid contexts.

### Core & Introduction
- [Core Directives](../../docs/directives/core.md) — `daemon`, `worker_processes`, `error_log`, `include`, `events` block
- [Intro & Guides](../../docs/directives/intro.md) — config units, connection methods, hashes, syslog, signals, QUIC, server names, request processing, load balancing, HTTPS setup

### HTTP Modules
- [HTTP Core](../../docs/directives/http/core.md) — `server`, `location`, `listen`, `root`, `index`, `try_files`, `alias`, `types`, `keepalive_timeout`, `client_max_body_size`
- [HTTP Proxy](../../docs/directives/http/proxy.md) — `proxy_pass`, `proxy_set_header`, `proxy_cache`, `proxy_cache_path`, `proxy_connect_timeout`, `proxy_read_timeout`
- [HTTP Upstream](../../docs/directives/http/upstream.md) — `upstream`, `server`, `keepalive`, `ip_hash`, `least_conn`, `hash`, `zone`
- [HTTP SSL](../../docs/directives/http/ssl.md) — `ssl_certificate`, `ssl_certificate_key`, `ssl_protocols`, `ssl_ciphers`, `ssl_session_cache`, `ssl_stapling`
- [HTTP Rewrite](../../docs/directives/http/rewrite.md) — `rewrite`, `return`, `if`, `set`, `break`, `rewrite_log`
- [HTTP Log](../../docs/directives/http/log.md) — `access_log`, `log_format`, `open_log_file_cache`
- [HTTP Limit Req](../../docs/directives/http/limit_req.md) — `limit_req_zone`, `limit_req`, burst, nodelay, `limit_req_status`
- [HTTP Limit Conn](../../docs/directives/http/limit_conn.md) — `limit_conn_zone`, `limit_conn`, `limit_conn_status`
- [HTTP Headers](../../docs/directives/http/headers.md) — `add_header`, `add_trailer`, `expires`
- [HTTP Access](../../docs/directives/http/access.md) — `allow`, `deny`
- [HTTP Auth Basic](../../docs/directives/http/auth_basic.md) — `auth_basic`, `auth_basic_user_file`
- [HTTP Auth Request](../../docs/directives/http/auth_request.md) — `auth_request`, `auth_request_set`
- [HTTP Geo](../../docs/directives/http/geo.md) — `geo` (IP-to-variable mapping)
- [HTTP GeoIP](../../docs/directives/http/geoip.md) — `geoip_country`, `geoip_city`
- [HTTP Map](../../docs/directives/http/map.md) — `map`
- [HTTP FastCGI](../../docs/directives/http/fastcgi.md) — `fastcgi_pass`, `fastcgi_param`, `fastcgi_cache`, `fastcgi_read_timeout`
- [HTTP gRPC](../../docs/directives/http/grpc.md) — `grpc_pass`, `grpc_bind`, `grpc_ssl_*`
- [HTTP GZip](../../docs/directives/http/gzip.md) — `gzip`, `gzip_comp_level`, `gzip_types`, `gzip_min_length`
- [HTTP RealIP](../../docs/directives/http/realip.md) — `set_real_ip_from`, `real_ip_header`, `real_ip_recursive`
- [HTTP Referer](../../docs/directives/http/referer.md) — `valid_referers`
- [HTTP v2](../../docs/directives/http/v2.md) — `http2`, `http2_push`, `http2_max_concurrent_streams`
- [HTTP v3](../../docs/directives/http/v3.md) — `http3`, `quic_retry`, `quic_gso`
- [HTTP SSI](../../docs/directives/http/ssi.md) — `ssi`, `ssi_types`
- [HTTP Sub](../../docs/directives/http/sub.md) — `sub_filter`, `sub_filter_once`
- [HTTP Slice](../../docs/directives/http/slice.md) — `slice` (byte-range caching)
- [HTTP Mirror](../../docs/directives/http/mirror.md) — `mirror`, `mirror_request_body`
- [HTTP Split Clients](../../docs/directives/http/split_clients.md) — `split_clients` (A/B testing)
- [HTTP SCGI](../../docs/directives/http/scgi.md) — `scgi_pass`, `scgi_param`
- [HTTP uWSGI](../../docs/directives/http/uwsgi.md) — `uwsgi_pass`, `uwsgi_param`
- [HTTP Upstream HC](../../docs/directives/http/upstream_hc.md) — `health_check`, `match`
- [HTTP JS (njs)](../../docs/directives/http/js.md) — `js_import`, `js_content`, `js_set`
- [HTTP Secure Link](../../docs/directives/http/secure_link.md) — `secure_link`, `secure_link_md5`
- [HTTP Stub Status](../../docs/directives/http/stub_status.md) — `stub_status`
- [HTTP Charset](../../docs/directives/http/charset.md) — `charset`, `source_charset`
- [HTTP DAV](../../docs/directives/http/dav.md) — `dav_methods`, `dav_access`
- [HTTP Image Filter](../../docs/directives/http/image_filter.md) — `image_filter`
- [HTTP Memcached](../../docs/directives/http/memcached.md) — `memcached_pass`
- [HTTP UserID](../../docs/directives/http/userid.md) — `userid`, `userid_name`, `userid_expires`

### Stream Modules (TCP/UDP)
- [Stream Core](../../docs/directives/stream/core.md) — `stream`, `server`, `listen`, `resolver`
- [Stream Proxy](../../docs/directives/stream/proxy.md) — `proxy_pass`, `proxy_bind`, `proxy_connect_timeout`, `proxy_timeout`, `proxy_ssl`
- [Stream SSL](../../docs/directives/stream/ssl.md) — `ssl_certificate`, `ssl_protocols`, `ssl_verify_client`
- [Stream SSL Preread](../../docs/directives/stream/ssl_preread.md) — `ssl_preread` (SNI routing)
- [Stream Upstream](../../docs/directives/stream/upstream.md) — `upstream`, `server`, `zone`, `hash`, `least_conn`
- [Stream Upstream HC](../../docs/directives/stream/upstream_hc.md) — `health_check`, `match`
- [Stream Limit Conn](../../docs/directives/stream/limit_conn.md) — `limit_conn_zone`, `limit_conn`
- [Stream Log](../../docs/directives/stream/log.md) — `access_log`, `log_format`
- [Stream Map](../../docs/directives/stream/map.md) — `map`
- [Stream Geo](../../docs/directives/stream/geo.md) — `geo`
- [Stream Access](../../docs/directives/stream/access.md) — `allow`, `deny`
- [Stream JS (njs)](../../docs/directives/stream/js.md) — `js_import`, `js_access`, `js_filter`, `js_preread`
- [Stream Return](../../docs/directives/stream/return.md) — `return`

### Mail Modules (SMTP/IMAP/POP3)
- [Mail Core](../../docs/directives/mail/core.md) — `mail`, `server`, `listen`, `protocol`
- [Mail SSL](../../docs/directives/mail/ssl.md) — `ssl_certificate`, `ssl_protocols`, `starttls`
- [Mail Auth HTTP](../../docs/directives/mail/auth_http.md) — `auth_http`, `auth_http_header`
- [Mail Proxy](../../docs/directives/mail/proxy.md) — `proxy_buffer`, `proxy_timeout`
- [Mail IMAP](../../docs/directives/mail/imap.md) — `imap_auth`, `imap_capabilities`
- [Mail POP3](../../docs/directives/mail/pop3.md) — `pop3_auth`, `pop3_capabilities`
- [Mail SMTP](../../docs/directives/mail/smtp.md) — `smtp_auth`, `smtp_capabilities`, `smtp_greeting_delay`

### Other
- [OpenTelemetry](../../docs/directives/otel.md) — `otel_exporter`, `otel_service_name`, `otel_trace`
- [Google PerfTools](../../docs/directives/google_perftools.md) — `google_perftools_profiles`
- [Full Index](../../docs/directives/README.md) — complete module listing with quick-reference configs
