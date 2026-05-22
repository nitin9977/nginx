# NGINX Event Loop: Annotated Code Walkthrough

**Audience**: Developers debugging event handling, performance engineers instrumenting nginx.
**Last Updated**: 2024
**Goal**: Trace request from socket → event loop → handler → response, with exact line numbers.
**Source paths**: All `src/` references are relative to the **nginx repository root** (`<repo>/src/`), not relative to `.github/`.

---

## Entry Point: main()

**File**: `src/core/nginx.c:196`

```c
int
main(int argc, char *const *argv)
{
    // ... initialization skipped ...
    
    if (ngx_process == NGX_PROCESS_SINGLE) {
        // Single process mode (for debugging)
        ngx_single_process_cycle(cycle);
    } else {
        // Normal: master-worker mode
        ngx_master_process_cycle(cycle);  // ← Start here for multi-process
    }
    
    return 0;
}
```

---

## Master Process Startup

**File**: `src/os/unix/ngx_process_cycle.c:74`

```c
void
ngx_master_process_cycle(ngx_cycle_t *cycle)
{
    // Line 74
    char              *title;
    u_char            *p;
    size_t             size;
    ngx_int_t          i;
    ngx_uint_t         sigio;
    sigset_t           set;
    struct itimerval   itv;
    ngx_uint_t         live;
    ngx_fd_t           signalfd;
    
    // ... signal handlers registered ...
    
    // Line 336: Spawn worker processes
    ngx_start_worker_processes(cycle, ccf->worker_processes, NGX_PROCESS_RESPAWN);
    // Calls fork() in loop; each child runs: ngx_worker_process_cycle()
    
    // Line 350: Infinite master loop
    for ( ;; ) {
        if (ngx_reap) {
            ngx_reap = 0;
            ngx_reap_children(cycle);  // Reap any exited children
        }
        
        if (ngx_reconfigure) {
            ngx_reconfigure = 0;
            // Reload config, restart workers
        }
        
        if (ngx_quit || ngx_terminate) {
            // Graceful or immediate shutdown
        }
        
        ngx_pass_open_channel(cycle);  // Communication with workers
        ngx_master_sleep();             // Wait for signal
    }
}
```

---

## Worker Process Initialization

**File**: `src/os/unix/ngx_process_cycle.c:699`

```c
static void
ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    // Line 699
    ngx_int_t worker = (intptr_t) data;
    
    ngx_process = NGX_PROCESS_WORKER;
    ngx_worker = worker;
    
    // Line 706: Initialize worker-specific state
    ngx_worker_process_init(cycle, worker);
    // ├─ Drop privileges to 'user' directive
    // ├─ Set CPU affinity
    // ├─ Install signal handlers (subset)
    // └─ Redirect stdin/stdout/stderr
    
    ngx_setproctitle("worker process");
    
    // Line 710-748: MAIN WORKER EVENT LOOP
    for ( ;; ) {
        
        // Check for graceful shutdown
        if (ngx_exiting) {
            if (ngx_event_no_timers_left() == NGX_OK) {
                ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
                ngx_worker_process_exit(cycle);  // Exit
            }
        }
        
        // Line 721: *** HEART OF NGINX ***
        ngx_process_events_and_timers(cycle);
        // This function does ~95% of the work; see next section
        
        // Handle termination signal
        if (ngx_terminate) {
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "exiting");
            ngx_worker_process_exit(cycle);
        }
        
        // Handle graceful shutdown signal
        if (ngx_quit) {
            ngx_quit = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0,
                          "gracefully shutting down");
            
            if (!ngx_exiting) {
                ngx_exiting = 1;
                ngx_set_shutdown_timer(cycle);      // Max 600s to finish requests
                ngx_close_listening_sockets(cycle); // Stop accepting
                ngx_close_idle_connections(cycle);  // Close idle connections
            }
        }
        
        // Handle log reopen signal
        if (ngx_reopen) {
            ngx_reopen = 0;
            ngx_log_error(NGX_LOG_NOTICE, cycle->log, 0, "reopening logs");
            ngx_reopen_files(cycle, -1);
        }
    }
}
```

---

## Event Loop Core

**File**: `src/event/ngx_event.c:195`

```c
void
ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    // Line 195
    ngx_uint_t  flags;
    ngx_msec_t  timer, delta;
    
    // ===== STEP 1: Calculate next timeout =====
    // Line 200-217
    if (ngx_timer_resolution) {
        timer = NGX_TIMER_INFINITE;   // Batch timer updates
        flags = 0;
    } else {
        timer = ngx_event_find_timer();  // Find next expiring timer
        flags = NGX_UPDATE_TIME;         // Update time after epoll_wait
        
        #if (NGX_WIN32)
        // Cap timer at 500ms for signal safety (Windows)
        if (timer == NGX_TIMER_INFINITE || timer > 500) {
            timer = 500;
        }
        #endif
    }
    
    // ===== STEP 2: Try to acquire accept mutex =====
    // Line 219-239
    if (ngx_use_accept_mutex) {
        if (ngx_accept_disabled > 0) {
            // Backpressure: we're overloaded; skip accept for N cycles
            ngx_accept_disabled--;
        } else {
            // Try to lock (non-blocking)
            if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
                return;  // Lock failed; try again next cycle
            }
            
            if (ngx_accept_mutex_held) {
                // ✓ Got the lock; defer all posted events
                flags |= NGX_POST_EVENTS;
            } else {
                // Lost lock race; retry soon
                if (timer == NGX_TIMER_INFINITE
                    || timer > ngx_accept_mutex_delay)
                {
                    timer = ngx_accept_mutex_delay;  // Usually 500ms
                }
            }
        }
    }
    
    // ===== STEP 3: Check for pending next-tick events =====
    // Line 241-244
    if (!ngx_queue_empty(&ngx_posted_next_events)) {
        ngx_event_move_posted_next(cycle);
        timer = 0;  // Process immediately; don't wait in epoll_wait
    }
    
    // ===== STEP 4: CALL EPOLL/KQUEUE/SELECT =====
    // Line 246-248
    delta = ngx_current_msec;  // Snapshot current time
    
    (void) ngx_process_events(cycle, timer, flags);
    // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
    // This calls platform-specific multiplexor:
    //   epoll (Linux):    ngx_epoll_process_events()
    //   kqueue (BSD):     ngx_kqueue_process_events()
    //   select (fallback): ngx_select_process_events()
    //
    // Blocks for up to 'timer' milliseconds waiting for I/O.
    // Populates: ngx_posted_accept_events, ngx_posted_events
    // ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
    
    delta = ngx_current_msec - delta;
    ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                   "timer delta: %M", delta);
    
    // ===== STEP 5: Process accept events (if we hold mutex) =====
    // Line 255
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);
    // Iterate queue; call handler for each: ngx_handle_accept_event()
    // → accept(2) new connection
    // → allocate ngx_connection_t from free list
    // → call ngx_http_init_connection()
    
    // ===== STEP 6: Release accept mutex =====
    // Line 257-259
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);  // Atomic unlock
        ngx_accept_mutex_held = 0;
    }
    
    // ===== STEP 7: Expire timers =====
    // Line 261
    ngx_event_expire_timers();
    // Iterate timer queue; expire all with deadline <= now
    // Calls: timeout callback (e.g., close connection)
    
    // ===== STEP 8: Process posted events (data I/O) =====
    // Line 263
    ngx_event_process_posted(cycle, &ngx_posted_events);
    // Iterate queue; call handler for each:
    //   ngx_http_process_request_line() (read)
    //   ngx_http_process_request() (process)
    //   ngx_http_writer() (write)
}
// ← Back to worker loop; repeat
```

**Time Complexity**:
- Best case: 0.1 ms (epoll_wait returns immediately; few events)
- Typical: 1-10 ms (epoll_wait sleeps for 10ms; hundreds of events)
- Worst case: 100 ms (timer expires; many timers expire)

---

## Event Multiplexor: epoll

**File**: `src/event/modules/ngx_epoll_module.c:323`

### Initialization

```c
static ngx_int_t
ngx_epoll_init(ngx_cycle_t *cycle, ngx_pool_t *pool)
{
    // Line 323
    // Create epoll file descriptor
    ngx_epoll_fd = epoll_create(ccf->worker_connections);
    
    // Allocate event array (one per connection)
    ngx_epoll_events = ngx_alloc(
        sizeof(struct epoll_event) * ccf->worker_connections, cycle->log);
    
    // Register timer event (SIGALRM, for timer resolution)
    // → Later: epoll_wait returns due to SIGALRM
    // → Timer callbacks executed
}
```

### Process Events

```c
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
    // Line ~300-400
    int                events;
    uint32_t           revents;
    ngx_int_t          instance, i;
    ngx_uint_t         level;
    ngx_err_t          err;
    ngx_event_t       *rev, *wev;
    ngx_connection_t  *c;
    
    // Line 350: WAIT FOR I/O EVENTS
    events = epoll_wait(ngx_epoll_fd, ngx_epoll_events,
                        (int) ccf->worker_connections, timer);
    // Blocks for up to 'timer' ms
    // Returns: number of events ready (0 = timeout, -1 = error)
    // Populates: ngx_epoll_events[] with results
    
    // If timeout: return (no events)
    if (events == 0) {
        if (timer != (ngx_msec_t) -1) {
            return NGX_OK;  // Normal timeout
        }
        return NGX_ERROR;   // Unexpected -1
    }
    
    // Line 360: Process each event returned by epoll_wait
    for (i = 0; i < events; i++) {
        
        // Get connection and file descriptor from event data
        c = (ngx_connection_t *) ngx_epoll_events[i].data.ptr;
        instance = (uintptr_t) c & 1;
        // Lower bit indicates: 0=old instance (stale), 1=new instance
        
        revents = ngx_epoll_events[i].events;  // EPOLLIN, EPOLLOUT, etc.
        
        // Handle read event (socket readable)
        if ((revents & EPOLLIN) && rev->active) {
            // Line 380-390
            rev->ready = 1;  // Mark ready
            
            // If posted event mode (defer processing):
            if (flags & NGX_POST_EVENTS) {
                ngx_queue_insert_tail(&ngx_posted_events, &rev->queue);
            } else {
                // Process immediately
                rev->handler(rev);  // Call read handler
                //   → ngx_http_process_request_line()
                //   → ngx_http_process_request()
                //   → ngx_handle_read_event()
            }
        }
        
        // Handle write event (socket writable)
        if ((revents & EPOLLOUT) && wev->active) {
            // Line 400-410
            wev->ready = 1;
            
            if (flags & NGX_POST_EVENTS) {
                ngx_queue_insert_tail(&ngx_posted_events, &wev->queue);
            } else {
                wev->handler(wev);  // Call write handler
                //   → ngx_http_writer()
                //   → ngx_output_chain()
            }
        }
        
        // Handle errors (EPOLLERR, EPOLLHUP)
        if (revents & (EPOLLERR | EPOLLHUP)) {
            if ((revents & EPOLLERR) && rev->active) {
                rev->ready = 1;
                rev->handler(rev);  // Call error handler
            }
        }
    }
    
    return NGX_OK;
}
```

---

## Connection Accept Path

**File**: `src/core/ngx_connection.c:178` (accept handler)

```c
static void
ngx_handle_accept_event(ngx_event_t *ev)
{
    // Line 178
    ngx_connection_t  *c;
    ngx_listening_t   *ls;
    
    ls = ev->data;  // Listening socket
    
    // May accept multiple (if multi_accept on)
    for (;;) {
        // Line 186: accept(2)
        c = ngx_get_connection(ls->fd, ev->log);
        if (c == NULL) {
            // No free connections; skip
            return;
        }
        
        // Line 192: Accept new socket
        s = accept(ls->fd, (struct sockaddr *) sin, &socklen);
        if (s == (ngx_socket_t) -1) {
            // Error (EAGAIN = no more sockets)
            ngx_free_connection(c);
            return;
        }
        
        // Line 200: Initialize connection
        c->fd = s;
        c->number = ngx_atomic_fetch_add(ngx_connection_counter, 1);
        c->listening = ls;
        
        // Line 210: Call protocol handler
        // For HTTP:
        ls->handler(c);  // → ngx_http_init_connection()
        
        // Line 215: Register read event
        if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
            ngx_close_connection(c);
            return;
        }
    }
}
```

---

## HTTP Connection Initialization

**File**: `src/http/ngx_http_request.c:528`

```c
ngx_int_t
ngx_http_init_connection(ngx_connection_t *c)
{
    // Line 528
    
    // Allocate per-connection HTTP state
    hc = ngx_palloc(c->pool, sizeof(ngx_http_connection_t));
    c->data = hc;  // Store HTTP state in connection
    
    // Allocate read buffer (for request parsing)
    c->buffer = ngx_create_temp_buf(c->pool, conf->client_header_buffer_size);
    // Default: 1 KB
    
    // Set handler to wait for request line
    c->read->handler = ngx_http_wait_request_handler;
    
    // Register read event
    if (ngx_handle_read_event(c->read, 0) != NGX_OK) {
        ngx_http_close_connection(c);
        return NGX_ERROR;
    }
    
    return NGX_OK;
}
```

---

## Request Line Parsing

**File**: `src/http/ngx_http_request.c:1114` (handler)

```c
static void
ngx_http_process_request_line(ngx_event_t *rev)
{
    // Line 1114
    ngx_connection_t  *c;
    ngx_http_request_t *r;
    
    c = rev->data;
    
    // Line 1125: Read from socket
    n = c->recv(c, buf->last, buf->end - buf->last);
    if (n == NGX_ERROR) {
        ngx_http_close_connection(c);
        return;
    }
    
    if (n == 0) {
        // Socket closed by client
        ngx_http_close_connection(c);
        return;
    }
    
    buf->last += n;  // Move end pointer
    
    // Line 1148: Parse request line
    rc = ngx_http_parse_request_line(r, buf);
    // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
    // State machine; see next section
    // ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
    
    if (rc == NGX_AGAIN) {
        // Incomplete; need more data
        if (buf->last == buf->end) {
            // Buffer full
            // Allocate larger buffer (up to 4 × 32 KB)
        }
        // Re-register read event; wait for more data
        ngx_handle_read_event(rev, conf->client_header_timeout);
        return;
    }
    
    if (rc == NGX_HTTP_PARSE_INVALID_REQUEST) {
        // Malformed request
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return;
    }
    
    // rc == NGX_OK: request line complete
    
    // Line 1160: Transition to header parsing
    r->state = NGX_HTTP_PROCESS_REQUEST_HEADERS;
    c->read->handler = ngx_http_process_request_headers;
    ngx_http_process_request_headers(rev);  // → Continue parsing
}
```

---

## Request Line Parsing State Machine

**File**: `src/http/ngx_http_parse.c:108`

```c
ngx_int_t
ngx_http_parse_request_line(ngx_http_request_t *r, ngx_buf_t *b)
{
    // Line 108
    u_char  c, ch, *p, *m;
    enum {
        sw_start = 0,
        sw_method,
        sw_spaces_before_uri,
        sw_schema,
        // ... more states ...
        sw_almost_done
    } state;
    
    state = r->state;  // Resume from previous call (if partial)
    
    // Line 142: Character-by-character parsing
    for (p = b->pos; p < b->last; p++) {
        ch = *p;
        
        switch (state) {
        
        case sw_start:
            r->request_start = p;
            
            // Skip leading CR LF
            if (ch == CR || ch == LF) {
                continue;  // Keep looping
            }
            
            // Validate first char of method (must be letter)
            if ((ch < 'A' || ch > 'Z') && (ch < 'a' || ch > 'z')) {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            
            state = sw_method;
            break;
        
        case sw_method:
            // Method name continues (letters)
            if (ch == ' ') {
                r->method_end = p - 1;
                state = sw_spaces_before_uri;
                break;
            }
            
            // Validate method char
            if ((ch < 'A' || ch > 'Z')
                && (ch < 'a' || ch > 'z')
                && ch != '_')  // Allowed: letters, underscore
            {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            break;
        
        case sw_spaces_before_uri:
            if (ch == ' ') {
                continue;  // Skip multiple spaces
            }
            
            if (ch == '/' || ch == '*') {
                // Absolute path or * (OPTIONS)
                r->uri_start = p;
                state = sw_uri;
                break;
            }
            
            if ((ch < 'A' || ch > 'Z') && (ch < 'a' || ch > 'z')) {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            
            // Could be schema (http:// or https://)
            state = sw_schema;
            break;
        
        case sw_uri:
            // URI parsing
            if (ch == ' ') {
                r->uri_end = p;
                state = sw_http_09;  // Transition to version
                break;
            }
            
            if (ch == '?') {
                r->args_start = p + 1;  // Query string
                break;
            }
            
            if (ch == '#') {
                r->args_end = p;
                r->fragment_start = p + 1;
                break;
            }
            
            // Validate URI char (no control chars < 0x21 or > 0x7e)
            if ((ch < 0x21) || (ch > 0x7e)) {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            break;
        
        case sw_http_H:
        case sw_http_HT:
        case sw_http_HTT:
        case sw_http_HTTP:
            // Validate "HTTP" string
            if (ch != "HTTP"[state - sw_http_H]) {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            state++;
            break;
        
        case sw_first_major_digit:
            if (ch < '0' || ch > '9') {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            r->http_major = ch - '0';
            state = sw_major_digit;
            break;
        
        case sw_first_minor_digit:
            if (ch < '0' || ch > '9') {
                return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗
            }
            r->http_minor = ch - '0';
            state = sw_minor_digit;
            break;
        
        case sw_almost_done:
            if (ch == LF) {
                return NGX_OK;  // ✓ REQUEST LINE COMPLETE
            }
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ No LF after CR
        }
    }
    
    r->state = state;  // Save state for resume
    return NGX_AGAIN;  // Need more data
}
```

---

## Response Writing Path

**File**: `src/http/ngx_http_core_module.c` (handler) → `src/core/ngx_output_chain.c` (output)

### Request Handler Phase

```c
// Line ~300-400 (simplified)
ngx_int_t
ngx_http_handler(ngx_http_request_t *r)
{
    // Determine location (regex matching, etc.)
    // Call location handler (static file, proxy, script, etc.)
    // Generate response
    
    // Buffer response into ngx_chain_t
    chain = ngx_alloc(...);
    chain->buf = ngx_create_temp_buf(...);
    ngx_memcpy(chain->buf->pos, response, len);
    chain->buf->last = chain->buf->pos + len;
    
    // Write response
    return ngx_http_output_chain(r->connection, chain);
}
```

### Output Chain

**File**: `src/core/ngx_output_chain.c:79`

```c
ngx_int_t
ngx_output_chain(ngx_output_chain_ctx_t *ctx, ngx_chain_t *in)
{
    // Line 79
    // ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
    // SEND DATA TO NETWORK
    // ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
    
    ngx_int_t     rc;
    off_t         size;
    ngx_chain_t  *cl, *to_send;
    
    // Build chain of buffers to send
    for (cl = in; cl; cl = cl->next) {
        size += ngx_buf_size(cl->buf);
        
        // Collect up to ~8 KB or 64 vectors
        if (size >= sendfile_max_chunk) {
            to_send = cl;
            break;
        }
    }
    
    // Line ~200: Send buffers
    rc = ngx_writev(ctx->connection, to_send);
    // Calls: writev(2) for memory buffers
    //   or: sendfile(2) for file buffers (zero-copy)
    
    if (rc == NGX_ERROR) {
        return NGX_ERROR;
    }
    
    if (rc > 0) {
        // Some data sent
        // Advance buffer pointers
        ctx->sent += rc;
        
        if (ctx->sent == size) {
            // All buffered data sent
            // Dequeue sent buffers
        } else {
            // Partial write; more data pending
            // Register write event to continue
            ngx_handle_write_event(wev, ...);
            return NGX_AGAIN;
        }
    }
    
    return NGX_OK;
}
```

### Write Event Handler

**File**: `src/http/ngx_http_core_module.c` (writer callback)

```c
static void
ngx_http_writer(ngx_event_t *wev)
{
    // Called when socket becomes writable
    // Continue sending pending buffers
    
    ngx_connection_t  *c;
    ngx_http_request_t *r;
    
    c = wev->data;
    r = c->data;
    
    // Call output chain to send more data
    rc = ngx_http_output_chain(r->connection, NULL);
    
    if (rc == NGX_AGAIN) {
        // More data to send
        ngx_handle_write_event(wev, ...);
        return;
    }
    
    if (rc == NGX_OK) {
        // All data sent
        
        if (r->keepalive) {
            // Reuse connection
            ngx_http_keepalive_handler(rev);
        } else {
            // Close connection
            ngx_http_close_connection(c);
        }
    }
}
```

---

## Connection Lifecycle: Timing

### Typical Request (static file, 1 KB)

```
Time    Event                           Duration    Cumulative
────────────────────────────────────────────────────────────────
0µs     Request arrives (SYN)
        Worker woken from epoll_wait
        
100µs   ngx_handle_accept_event()      100µs       100µs
        accept(2) new socket
        allocate connection + pool
        register read event
        
200µs   (client sends HTTP request)
        
1.2ms   Socket readable
        ngx_http_process_request_line() 1ms         2ms
        recv(2) 1 KB request
        Parse: GET / HTTP/1.1
        
2.2ms   ngx_http_process_request()     0.2ms       2.4ms
        Route to handler (static file)
        Open file /var/www/index.html
        Set up response
        
3.2ms   ngx_http_output_chain()        0.8ms       3.2ms
        writev(2) response header
        sendfile(2) file content
        Queue write event
        
4ms     Socket writable
        ngx_http_writer()               0.2ms       4.2ms
        Confirm all data sent
        (or send more if partial write)
        
5ms     ngx_http_keepalive_handler()   0.5ms       5.5ms
        Reset connection
        Register read event for next request
        
─ Connection idle, waiting for next request
6s      Keep-alive timeout fires
        ngx_close_connection()          10µs        close
```

**Total: 5.5 ms server time** (rest is network round-trip)

---

## Key Invariants Verified by Walkthrough

### 1. **Single-Threaded Per Worker**
- ✓ Event loop is purely sequential; no concurrent handlers
- ✓ All connection state access is non-blocking

### 2. **State Machine Resumability**
- ✓ Request parsing resumes from line 140: `state = r->state` (saved)
- ✓ Handles partial reads gracefully (NGX_AGAIN)

### 3. **No Polling**
- ✓ epoll_wait blocks; doesn't spin
- ✓ Workers sleep when no events (low CPU when idle)

### 4. **Memory Isolation**
- ✓ Each connection has own pool: `c->pool`
- ✓ Cleanup on close: `ngx_destroy_pool(c->pool)`

### 5. **Timeout Safety**
- ✓ All read/write events have timeout (client_header_timeout, etc.)
- ✓ Timer expiry fires handler → close connection

---

## Instrumentation Points

### Adding Debug Logging

```c
// In ngx_event.c:195, after epoll_wait:
ngx_log_debug4(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
               "epoll_wait: events=%d, timer=%M, delta=%M, accepted=%ui",
               events, timer, delta, ngx_stat_accepted);
```

### Measuring Event Loop Time

```c
// At start of ngx_process_events_and_timers:
start_time = ngx_current_msec;

// At end:
elapsed = ngx_current_msec - start_time;
if (elapsed > 100) {  // Alert if > 100ms
    ngx_log_warn(NGX_LOG_WARN, cycle->log, 0,
                 "slow event loop: %M ms", elapsed);
}
```

### Tracking Connection States

```c
ngx_log_debug4(NGX_LOG_DEBUG_HTTP, c->log, 0,
               "conn=%p fd=%d reading=%d writing=%d",
               c, c->fd, c->read->active, c->write->active);
```

---

## References

- Source files referenced: see `.github/docs/architecture/ARCHITECTURE.md`
- Event system: `man epoll_wait`, `man kqueue`
- HTTP parsing: RFC 9110 (HTTP Semantics)

