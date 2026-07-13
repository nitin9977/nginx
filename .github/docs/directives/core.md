# nginx Core Functionality Directives

**Source**: https://nginx.org/en/docs/ngx_core_module.html  
**Module**: ngx_core_module (built-in)

---

## Main Context Directives

| Directive | Syntax | Default | Description |
|-----------|--------|---------|-------------|
| `daemon` | `on \| off` | `on` | Run nginx as a daemon. Set `off` for development. |
| `debug_points` | `abort \| stop` | — | Trigger core dump (`abort`) or stop (`stop`) on internal error. For debugging only. |
| `env` | `variable[=value]` | `env TZ` | Preserve/set environment variables for workers (used by live upgrade and Perl module). |
| `error_log` | `file [level] [json]` | `logs/error.log error` | Log file path and level (`debug`,`info`,`notice`,`warn`,`error`,`crit`,`alert`,`emerg`). Supports `stderr`, `syslog:`, `memory:`. `json` format available in commercial. |
| `include` | `file \| mask` | — | Include another config file or glob pattern. |
| `load_module` | `file` | — | Load a dynamic module (`.so`). Since 1.9.11. |
| `lock_file` | `file` | `logs/nginx.lock` | Prefix for lock file names (used on platforms without atomic ops). |
| `master_process` | `on \| off` | `on` | Start worker processes. Set `off` for nginx developers only. |
| `pcre_jit` | `on \| off` | `off` | Enable PCRE JIT compilation for faster regex. Requires PCRE ≥ 8.20 with `--enable-jit`. |
| `pid` | `file` | `logs/nginx.pid` | File storing the main process PID. |
| `ssl_engine` | `device` | — | Hardware SSL accelerator name. |
| `ssl_object_cache_inheritable` | `on \| off` | `on` | Inherit SSL objects (certs, keys, CRLs) across config reloads if file unchanged. Since 1.27.4. |
| `thread_pool` | `name threads=N [max_queue=N]` | `default threads=32 max_queue=65536` | Thread pool for async file I/O. Requires `--with-threads`. Since 1.7.11. |
| `timer_resolution` | `interval` | — | Reduce `gettimeofday()` frequency. Example: `timer_resolution 100ms`. |
| `user` | `user [group]` | `nobody nobody` | User/group credentials for worker processes. |
| `worker_cpu_affinity` | `cpumask ... \| auto [mask]` | — | Bind workers to CPUs. `auto` for automatic assignment. Linux/FreeBSD only. |
| `worker_priority` | `number` | `0` | Worker process nice level (negative = higher priority, range -20..20). |
| `worker_processes` | `number \| auto` | `1` | Number of worker processes. `auto` detects CPU count. |
| `worker_rlimit_core` | `size` | — | `RLIMIT_CORE` for core file size. |
| `worker_rlimit_nofile` | `number` | — | `RLIMIT_NOFILE` max open files per worker. |
| `worker_shutdown_timeout` | `time` | — | Grace period before force-closing connections on shutdown. Since 1.11.11. |
| `working_directory` | `directory` | — | Working directory for workers (core files written here). |

---

## Events Context Directives

| Directive | Syntax | Default | Description |
|-----------|--------|---------|-------------|
| `accept_mutex` | `on \| off` | `off` | Serialize accept() across workers. Not needed with EPOLLEXCLUSIVE or reuseport. |
| `accept_mutex_delay` | `time` | `500ms` | Wait time before a worker retries accept when mutex is held. |
| `debug_connection` | `address \| CIDR \| unix:` | — | Enable debug logging for specific client addresses. Requires `--with-debug`. |
| `multi_accept` | `on \| off` | `off` | Accept all pending connections at once (ignored with kqueue). |
| `stall_threshold` | `time` | `1000ms` | Report a stall when event loop iteration exceeds this time. Commercial only. Since 1.29.0. |
| `use` | `method` | — | Event processing method (`epoll`, `kqueue`, `eventport`, etc.). Usually auto-selected. |
| `worker_aio_requests` | `number` | `32` | Max outstanding async I/O operations per worker (epoll + aio). Since 1.1.4. |
| `worker_connections` | `number` | `512` | Max simultaneous connections per worker (all connections, not just client). |
