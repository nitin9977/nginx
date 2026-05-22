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
