# NGINX Event Module Ramp-Up Guide

**Audience**: Engineers onboarding to `src/event` internals  
**Scope**: Architecture, module integration, and code walkthrough for a concrete event path  
**Date**: 2026-05-19

---

## 1. Architecture and Design of `src/event`

### 1.1 What the event module is responsible for

The `src/event` subsystem is nginx's OS-abstraction layer for I/O readiness and timer dispatch.

It owns:

- Event method selection (`epoll`, `kqueue`, `eventport`, `poll`, `select`, `iocp`)
- Registration and deregistration of read/write interest per connection
- Timer storage and expiration callbacks
- Deferred event execution via posted queues
- Accept orchestration (including accept mutex and optional exclusive accept behavior)

It does **not** parse HTTP, route requests, or generate upstream traffic directly. It hands accepted and ready connections to protocol modules (`http`, `stream`, `mail`, `quic`) via connection handlers.

### 1.2 Core design objects

#### `ngx_event_t` (`src/event/ngx_event.h`)

`ngx_event_t` is the per-direction event state object bound to a connection (`read` and `write` events per `ngx_connection_t`).

Key fields to understand first:

- `data`: back-pointer to owner (usually `ngx_connection_t`)
- `handler`: callback invoked when this event is dispatched
- `active`, `ready`, `timedout`, `posted`, `timer_set`: scheduler and state-machine bits
- `instance`: stale-event guard used by epoll/kqueue paths to detect recycled fd events
- `timer` (rbtree node) and `queue` (posted queue node)

#### `ngx_event_actions_t` (`src/event/ngx_event.h`)

This is the backend vtable selected at runtime:

- `add/del/enable/disable`
- `add_conn/del_conn`
- `process_events`
- `init/done`

`ngx_event_actions` is populated by the selected event module (for example `ngx_epoll_module`).

### 1.3 Runtime control loop

Main worker loop calls `ngx_process_events_and_timers()` each iteration.

High-level algorithm:

1. Compute wait timeout from nearest timer (`ngx_event_find_timer()`) unless timer resolution overrides it.
2. Handle accept mutex policy and decide whether accept events should be posted.
3. Move `ngx_posted_next_events` into active posted queue if needed.
4. Invoke backend poller via `ngx_process_events(...)`.
5. Process posted accept events.
6. Release accept mutex if held.
7. Expire timers (`ngx_event_expire_timers()`), invoking timeout handlers.
8. Process normal posted events.

This ordering is intentional to avoid starvation and to control accept fairness under load.

### 1.4 Internal queues and timer model

#### Posted queues (`src/event/ngx_event_posted.c`, `src/event/ngx_event_posted.h`)

- `ngx_posted_accept_events`
- `ngx_posted_events`
- `ngx_posted_next_events`

`ngx_post_event()` marks `ev->posted = 1` and appends once. This prevents duplicate enqueueing of the same event object.

#### Timer rbtree (`src/event/ngx_event_timer.c`, `src/event/ngx_event_timer.h`)

- Global timer store is an rbtree keyed by absolute expiration msec.
- `ngx_event_add_timer()` performs lazy update optimization (`NGX_TIMER_LAZY_DELAY`) to reduce churn.
- `ngx_event_expire_timers()` pops expired minimum nodes and directly calls `ev->handler(ev)`.

Important implication: timer handler execution happens in the same worker thread/process context as I/O handlers.

### 1.5 Accept path architecture

`ngx_event_accept()` is the hot path for stream listeners.

It performs:

- `accept`/`accept4`
- EMFILE/ENFILE backpressure behavior (disable accept events, delay retries)
- Connection object allocation via `ngx_get_connection()`
- Per-connection pool/log/sockaddr initialization
- Nonblocking mode setup
- Handler handoff: `ls->handler(c)` to protocol layer

For UDP and QUIC listeners, different recv handlers are bound during process init.

### 1.6 Backend modularity design

The directory `src/event/modules` implements backend-specific mechanics, each providing actions through `ngx_event_actions_t`:

- Linux: `ngx_epoll_module.c`
- BSD/macOS: `ngx_kqueue_module.c`
- Solaris: `ngx_eventport_module.c`, `ngx_devpoll_module.c`
- Generic fallback: `ngx_poll_module.c`, `ngx_select_module.c`
- Windows: `ngx_iocp_module.c`, win32 poll/select variants

Backend modules are intentionally isolated from higher-level protocol code.

---

## 2. Integration and Dependencies

### 2.1 Startup and worker integration

Integration entry points:

- Worker process loop: `src/os/unix/ngx_process_cycle.c` calls `ngx_process_events_and_timers()`.
- Event process init: `ngx_event_process_init()` in `src/event/ngx_event.c` allocates event arrays and connection pool structures for each worker.

At init time it also:

- Selects configured backend (`use` directive)
- Initializes posted queues and timer system
- Binds accept handlers to each listening socket connection event

### 2.2 Config subsystem integration

`events { ... }` configuration is parsed by `ngx_events_block()` and per-event-module config is created and initialized.

Core directives in event core module:

- `worker_connections`
- `use`
- `multi_accept`
- `accept_mutex`
- `accept_mutex_delay`
- `debug_connection`

This wiring is in `src/event/ngx_event.c` and integrates with nginx core config machinery (`ngx_conf_parse`, module contexts).

### 2.3 Core connection subsystem integration (`src/core`)

Event module relies heavily on `src/core/ngx_connection.c`:

- `ngx_get_connection()` allocates a reusable connection slot and resets read/write `ngx_event_t` state.
- `ngx_close_connection()` removes active events (`ngx_del_conn` or `ngx_del_event`), removes timers, removes posted entries, and closes fd.
- `instance` bit toggling in `ngx_get_connection()` is essential for stale readiness suppression in epoll path.

### 2.4 Protocol layer integration (`http`, `stream`, `mail`, `quic`)

After accept, event layer dispatches to protocol-specific listener handler via `ls->handler(c)`.

Typical handlers:

- HTTP: `ngx_http_init_connection()`
- Stream: `ngx_stream_init_connection()`
- Mail: `ngx_mail_init_connection()`

This is the principal seam between transport readiness and protocol state machines.

### 2.5 OS/kernel integration

The event module depends on kernel APIs and behavior:

- Pollers: `epoll_wait`, kqueue kevents, poll/select, event ports, IOCP
- Socket controls: nonblocking mode, optional `SO_SNDLOWAT`
- Optional timer signal path with `setitimer(SIGALRM)` when backend does not provide native timer events

Runtime assumptions are backend-specific (for example epoll stale event handling and EPOLLEXCLUSIVE fairness behavior).

### 2.6 Optional dependency integrations

- OpenSSL hooks in `ngx_event_openssl*.c`
- UDP and QUIC paths in `ngx_event_udp.c` and `src/event/quic/*`
- Async file I/O integration in epoll path via `eventfd` and Linux AIO syscalls when enabled

---

## 3. Basic Code Walkthrough for a Given Event

This section gives a practical tracing template and one concrete example.

### 3.1 Generic tracing template (works for most events)

Given a specific ready event (read/write/timer), trace in this order:

1. Worker loop invokes `ngx_process_events_and_timers()`.
2. Selected backend `process_events` marks `ev->ready = 1` and either:
   - calls `ev->handler(ev)` immediately, or
   - posts it to a queue when `NGX_POST_EVENTS` is active.
3. Posted queue processor (`ngx_event_process_posted`) invokes handler.
4. Handler may:
   - consume I/O (`recv/send`),
   - add/remove timers,
   - re-arm interest (`ngx_handle_read_event` or `ngx_handle_write_event`),
   - transition protocol state via function pointer updates.

### 3.2 Concrete walkthrough: accept read event on Linux epoll

#### Step A: Worker blocks in poller

`ngx_process_events_and_timers()` computes timeout and calls `ngx_process_events` (epoll implementation: `ngx_epoll_process_events`).

#### Step B: epoll returns readiness

In `ngx_epoll_process_events`:

- Retrieve `c = event_list[i].data.ptr`
- Decode `instance` bit and reject stale recycled events
- If `EPOLLIN` and `rev->active`, set:
  - `rev->ready = 1`
  - `rev->available = -1`
- Dispatch immediately or queue with `ngx_post_event(rev, &ngx_posted_accept_events)` when posting is enabled

#### Step C: accept handler executes

`rev->handler` for listening stream sockets is set to `ngx_event_accept` during `ngx_event_process_init()`.

Inside `ngx_event_accept`:

1. Call `accept4`/`accept`
2. Handle transient and resource errors (`EAGAIN`, `ECONNABORTED`, `EMFILE`, `ENFILE`)
3. Acquire `ngx_connection_t` from free list (`ngx_get_connection`)
4. Create per-connection memory pool and log context
5. Set socket mode and inherited attributes
6. Initialize connection I/O function pointers (`recv/send` and chain variants)
7. Bind read/write event logs, flags, and counters
8. Call listener protocol handoff `ls->handler(c)`

#### Step D: protocol initialization continues

For HTTP listeners, `ls->handler` points to `ngx_http_init_connection`:

- Allocates HTTP connection context
- Resolves address-specific server configuration
- Attaches per-connection log context
- Sets initial HTTP state machine handlers

At this point event module has completed transport-level admission and protocol code owns higher-level request progression.

### 3.3 Secondary walkthrough: timer expiration event

Timer event path is independent of kernel readiness delivery:

1. Handler schedules timeout via `ngx_add_timer(ev, delta)`.
2. Event node is inserted into global timer rbtree with absolute expiry.
3. Next loop iteration computes sleep with `ngx_event_find_timer()`.
4. `ngx_event_expire_timers()` pops expired nodes, sets `ev->timedout = 1`, and calls `ev->handler(ev)`.

This is why most protocol handlers check `ev->timedout` first and branch into timeout logic.

---

## 4. Fast Mental Model for New Engineers

Use this simplified model while reading code:

- Each connection has two event state objects: read and write.
- Backend poller only reports readiness and basic edge metadata.
- Real behavior lives in function pointers (`ev->handler`, `ls->handler`).
- Posted queues are scheduling control, not persistence.
- Timers are first-class events stored in a global rbtree.
- Protocol modules own parsing and request state; event module owns readiness orchestration.

---

## 5. Recommended First Read Order in `src/event`

1. `src/event/ngx_event.h`
2. `src/event/ngx_event.c`
3. `src/event/ngx_event_accept.c`
4. `src/event/ngx_event_timer.c` and `src/event/ngx_event_timer.h`
5. `src/event/ngx_event_posted.c` and `src/event/ngx_event_posted.h`
6. Your platform backend in `src/event/modules` (usually `ngx_epoll_module.c` on Linux)
7. Handoff target in protocol module (`ngx_http_init_connection`, etc.)

---

## 6. Practical Debug Checklist

When behavior seems wrong for an event, inspect in this order:

1. Was the event registered (`active`) and not stale (`instance`)?
2. Did backend mark `ready` and choose immediate vs posted dispatch?
3. Did the event get dropped due to connection close and recycled fd?
4. Was timer state (`timer_set`, `timedout`) coherent?
5. Did handler re-arm read/write interest appropriately?
6. Did protocol handler replace callbacks in unexpected order?

This checklist catches most real-world regressions in event dispatch paths.
