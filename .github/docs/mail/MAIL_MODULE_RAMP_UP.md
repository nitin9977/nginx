# NGINX Mail Module Ramp-Up Guide

**Audience**: Engineers onboarding to `src/mail` internals  
**Scope**: Architecture, integration, and walkthrough for representative mail session flows  
**Date**: 2026-05-19  
**Source paths**: All `src/` references are relative to the **nginx repository root** (`<repo>/src/`), not relative to `.github/`.

---

## 1. Architecture and Design of `src/mail`

### 1.1 What `src/mail` does

`src/mail` implements stateful mail protocol frontends (POP3, IMAP, SMTP) with optional auth proxy and TLS support.

It owns:

- `mail {}` configuration block and module contexts
- Listener-to-server configuration mapping for mail ports
- Mail session object lifecycle
- Protocol command parsers and handler state machines for POP3/IMAP/SMTP
- Auth HTTP proxy integration (`ngx_mail_auth_http_module`)
- Upstream mail proxy relay (`ngx_mail_proxy_module`)

### 1.2 Top-level design

- Config assembly lives in `src/mail/ngx_mail.c`
- Connection/session bootstrap lives in `src/mail/ngx_mail_handler.c`
- Protocol-specific state machines are split per protocol: `ngx_mail_pop3_handler.c`, `ngx_mail_imap_handler.c`, `ngx_mail_smtp_handler.c`

Core architectural split:

1. **Build-time config graph**: `main_conf` + `srv_conf` contexts
2. **Runtime per-connection session graph**: `ngx_mail_session_t` + protocol handler pointers

### 1.3 Key design objects

#### `ngx_mail_session_t` (`src/mail/ngx_mail.h`)

The per-connection session state. Key fields:

- `signature`: magic `0x4C49414D` (`"MAIL"`) — used for type-safety checks
- `connection`: back-pointer to `ngx_connection_t`
- `out`: response string pending write
- `buffer`: input buffer for command parsing
- `ctx` / `main_conf` / `srv_conf`: module config context arrays
- `resolver_ctx`: DNS resolution context for auth
- `proxy`: pointer to `ngx_mail_proxy_ctx_t` (upstream relay state)
- `mail_state`: protocol-independent state index

**Bit-field flags** (protocol and auth state):

- `ssl:1`: SSL/TLS active
- `protocol:3`: protocol type (POP3=0, IMAP=1, SMTP=2)
- `blocked:1`: output blocked (flow control)
- `quit:1`: session terminating
- `quoted:1`, `backslash:1`: IMAP literal parser state
- `no_sync_literal:1`: IMAP non-synchronizing literal mode
- `starttls:1`: STARTTLS negotiation in progress
- `esmtp:1`: EHLO received (extended SMTP)
- `auth_method:3`: current auth mechanism enum
- `auth_wait:1`: waiting for auth HTTP response

**Auth and command fields**:

- `login` / `passwd`: extracted credentials (cleared after auth attempt)
- `salt`: CRAM-MD5 / APOP challenge string
- `tag` / `tagged_line`: IMAP command tag handling
- `smtp_helo` / `smtp_from` / `smtp_to`: SMTP envelope state
- `cmd` / `command` / `args`: parsed command, command code, argument array
- `errors` / `login_attempt`: error and auth failure counters

**Parser state**:

- `state`: current parser automaton state (protocol-specific enum value)
- `tag_start` / `cmd_start` / `arg_start`: character pointers into buffer
- `literal_len`: IMAP literal byte count

#### `ngx_mail_protocol_t` (`src/mail/ngx_mail.h`)

Protocol vtable — one per protocol (POP3, IMAP, SMTP):

- `name`: protocol name string
- `alpn`: ALPN identifier for TLS negotiation
- `port[4]`: default port numbers
- `type`: protocol type enum
- `init_session`: callback to send initial greeting
- `init_protocol`: callback to set read handler for command loop
- `parse_command`: command parser automaton
- `auth_state`: authentication state machine handler
- `internal_server_error` / `cert_error` / `no_cert`: canned error response strings

#### Protocol state enums

**POP3** (`ngx_pop3_state_e` — 8 states):
`start → user → passwd → auth_login_username → auth_login_password → auth_plain → auth_cram_md5 → auth_external`

**IMAP** (`ngx_imap_state_e` — 9 states):
`start → auth_login_username → auth_login_password → auth_plain → auth_cram_md5 → auth_external → login → user → passwd`

**SMTP** (`ngx_smtp_state_e` — 16 states):
`start → auth_login_username → auth_login_password → auth_plain → auth_cram_md5 → auth_external → helo → helo_xclient → helo_auth → helo_from → xclient → xclient_from → xclient_helo → xclient_auth → from → to`

These enums drive the parser's `s->mail_state` progression through command processing.

#### `ngx_mail_proxy_ctx_t` (`src/mail/ngx_mail.h`)

Upstream relay state:

- `upstream`: `ngx_peer_connection_t` to backend mail server
- `buffer`: I/O buffer for proxied data
- `proxy_protocol`: whether to send PROXY protocol header upstream

### 1.4 Configuration model

`ngx_mail_block()` in `src/mail/ngx_mail.c` creates module contexts and parses `mail {}`:

1. Initialize each mail module's `main_conf` and `srv_conf`
2. Call `preconfiguration` hooks
3. Parse `mail {}` block directives
4. Merge per-server `srv_conf` via `merge_srv_conf`
5. Build optimized port → address → server listener structures
6. Call `postconfiguration` hooks

Routing decisions are encoded into `ngx_mail_addr_conf_t` at config time, not per-request.

### 1.5 Session model

`ngx_mail_init_connection()` in `src/mail/ngx_mail_handler.c`:

1. Resolves effective server config from local `address:port`
2. Allocates `ngx_mail_session_t` in connection pool
3. Attaches `main_conf` / `srv_conf` pointers
4. Sets log context and initial read handler
5. Handles optional PROXY protocol preamble
6. On SSL listeners, performs SSL handshake first
7. Transitions into `ngx_mail_init_session()` which:
   - Sets `s->protocol` from `cscf->protocol->type`
   - Allocates per-module context array
   - Sets `c->write->handler = ngx_mail_send`
   - Calls `cscf->protocol->init_session(s, c)` — protocol-specific greeting

Design pattern: event read handler function pointers represent protocol/session state transitions.

---

## 2. Integration and Dependencies

### 2.1 Integration with `src/event`

Event module accepts connections and invokes listener callback. For mail listeners, callback is `ngx_mail_init_connection()`.

From there, mail module:

- Uses read event handlers for staged bootstrap (PROXY protocol → SSL → session init → command loop)
- Arms read timers via `ngx_add_timer()`
- Re-arms readiness interest with `ngx_handle_read_event()`
- Closes connection on timeout/errors or protocol completion

### 2.2 Integration with `src/core`

Mail relies on core for:

- `ngx_pool_t` — all session memory is pool-backed
- `ngx_connection_t` — transport endpoint
- `ngx_log_t` — error and debug logging
- `ngx_conf_t` / `ngx_command_t` — directive parsing
- `ngx_resolver_t` — DNS resolution for auth HTTP backend

Lifetime model: mail session memory is pool-backed and tied to connection lifecycle. `ngx_mail_close_connection()` destroys the pool after closing the socket.

### 2.3 Integration with protocol submodules

POP3/IMAP/SMTP handler modules implement command-specific logic after common session bootstrap.

Auxiliary integrations:

- `ngx_mail_auth_http_module`: external auth decisioning via HTTP subrequest
- `ngx_mail_proxy_module`: upstream relay/proxy behavior
- `ngx_mail_ssl_module`: TLS and certificate handling
- `ngx_mail_realip_module`: client address override from PROXY protocol

### 2.4 Auth flow integration

The auth path traverses multiple modules:

1. Protocol handler parses credentials → calls `ngx_mail_auth(s, c)`
2. `ngx_mail_auth()` resets parser state, increments `login_attempt`, calls `ngx_mail_auth_http_init(s)`
3. Auth HTTP module makes HTTP request to configured `auth_http` server
4. Response determines: allow (connect to backend), deny (send error), or wait

---

## 3. Basic Code Walkthrough for Mail Flows

### 3.1 Generic tracing template

1. Listener accepts → `ngx_mail_init_connection()`
2. Determine `addr_conf` and allocate `ngx_mail_session_t`
3. Set initial read handler and log action
4. If PROXY protocol enabled, consume/validate header first
5. If SSL, perform handshake
6. Enter `ngx_mail_init_session()` → protocol greeting
7. Command loop: protocol's `init_protocol` sets read handler
8. On timeout/error: close via `ngx_mail_close_connection()`

### 3.2 Concrete walkthrough: session initialization

#### Step A: connection handoff

`ngx_mail_init_connection()` (handler.c line ~165):

1. Looks up `ngx_mail_port_t` from `c->listening->servers`
2. Matches client address to find `ngx_mail_addr_conf_t`
3. Sets `cscf = addr_conf->default_server` (or virtual server)

#### Step B: session allocation

1. `s = ngx_pcalloc(c->pool, sizeof(ngx_mail_session_t))` — pool-backed
2. `s->signature = NGX_MAIL_MODULE` (the `"MAIL"` magic)
3. `s->main_conf = addr_conf->ctx->main_conf`
4. `s->srv_conf = addr_conf->ctx->srv_conf`
5. `c->data = s` — attach session to connection
6. Set `c->log->handler = ngx_mail_log_error`

#### Step C: SSL or init path

- If `addr_conf->ssl`: set `c->log->action = "SSL handshaking"`, call `ngx_mail_ssl_init_connection()`
- If PROXY protocol enabled: set `rev->handler = ngx_mail_proxy_protocol_handler`
- Otherwise: set `rev->handler = ngx_mail_init_session_handler`
- Arm read timer: `ngx_add_timer(rev, cscf->timeout)`

#### Step D: `ngx_mail_init_session()` (handler.c line ~460)

1. `s->protocol = cscf->protocol->type` — POP3/IMAP/SMTP
2. `s->ctx = ngx_pcalloc(c->pool, sizeof(void *) * ngx_mail_max_module)` — per-module context array
3. `c->write->handler = ngx_mail_send` — common write handler
4. `cscf->protocol->init_session(s, c)` — protocol sends greeting:
   - POP3: `+OK POP3 ready`
   - IMAP: `* OK IMAP4 ready`
   - SMTP: `220 hostname ESMTP ready`

### 3.3 Concrete walkthrough: auth flow

#### Step A: credential extraction

Protocol command handler (e.g., `ngx_mail_smtp_auth_state()`) parses AUTH command:

1. Extracts `s->login` and `s->passwd` from command arguments
2. Handles multi-step auth (LOGIN mechanism requires two round-trips)
3. Calls `ngx_mail_auth(s, c)`

#### Step B: `ngx_mail_auth()` (handler.c line ~910)

1. `s->args.nelts = 0` — clear argument array
2. Reset buffer: `s->buffer->pos = s->buffer->last = s->buffer->start`
3. `s->state = 0` — reset parser state
4. Delete read timer if set
5. `s->login_attempt++` — increment failure counter
6. Call `ngx_mail_auth_http_init(s)` — delegate to auth HTTP module

#### Step C: auth HTTP decision

1. Auth HTTP module sends HTTP request to configured backend
2. Backend response headers encode: Auth-Status, Auth-Server, Auth-Port, Auth-User
3. On success: `ngx_mail_proxy_init()` connects to upstream mail server
4. On failure: send protocol-appropriate error, increment `s->errors`
5. If `s->errors >= cscf->max_errors`: close connection

### 3.4 Concrete walkthrough: connection close

`ngx_mail_close_connection()` (handler.c line ~950):

1. If SSL active: call `ngx_ssl_shutdown(c)` — may return `NGX_AGAIN` for async
2. Decrement `ngx_stat_active` (stat stub)
3. `c->destroyed = 1`
4. Save `pool = c->pool`
5. `ngx_close_connection(c)` — removes events, closes fd, returns slot to free list
6. `ngx_destroy_pool(pool)` — frees all session memory, runs cleanup handlers

---

## 4. Fast Mental Model for New Engineers

- **Session = connection state**: `ngx_mail_session_t` is the per-connection control block
- **Protocol vtable**: `ngx_mail_protocol_t` dispatches to protocol-specific init/parse/auth handlers
- **State machine progression**: `s->mail_state` advances through protocol-specific enum states
- **Handler pointers = state**: the current read handler function pointer IS the session state
- **Auth is external**: credentials are validated by HTTP subrequest, not by nginx itself
- **Proxy relay**: after successful auth, upstream proxy module takes over bidirectional data transfer
- **Everything dies with the pool**: `ngx_mail_close_connection()` destroys connection pool, freeing all session memory

---

## 5. Recommended First Read Order in `src/mail`

1. `src/mail/ngx_mail.h` — all struct definitions, protocol enums, session fields
2. `src/mail/ngx_mail_handler.c` — session bootstrap, init, auth dispatch, close
3. `src/mail/ngx_mail.c` — config block parsing and listener setup
4. `src/mail/ngx_mail_core_module.c` — core module directives and server config
5. `src/mail/ngx_mail_smtp_handler.c` — SMTP protocol state machine (most complex)
6. `src/mail/ngx_mail_auth_http_module.c` — auth HTTP request/response flow
7. `src/mail/ngx_mail_proxy_module.c` — upstream proxy relay logic

---

## 6. Practical Debug Checklist

When debugging mail module behavior:

1. Did `addr_conf->server` mapping choose the expected `srv_conf`? Check listener address matching.
2. Is PROXY protocol enabled and parsing succeeding? Check `c->read->handler` after accept.
3. Is SSL handshake completing? Check for `ngx_mail_ssl_init_connection()` errors.
4. What is `s->mail_state`? Map to protocol enum to identify current command processing stage.
5. What is the current `c->read->handler`? This IS the session state — trace to see which function runs next.
6. Are timeout timers set/cleared consistently? Check `ngx_add_timer` / `ngx_del_timer` pairing.
7. Did auth HTTP request succeed? Check Auth-Status header in response.
8. Is `s->errors` approaching `cscf->max_errors`? Could trigger premature disconnect.
9. Is `s->login_attempt` count expected? Each `ngx_mail_auth()` call increments it.
10. Did `ngx_mail_close_connection()` reach pool destruction? SSL shutdown may defer close asynchronously.
