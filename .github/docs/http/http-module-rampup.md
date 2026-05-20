# nginx `src/http` Module — Engineering Ramp-Up Guide

> Principal-engineer depth. Every claim is traceable to a source file and line.  
> Target audience: engineers new to the nginx HTTP codebase.

---

## Table of Contents

1. [Architecture & Design](#1-architecture--design)
2. [Module Integration & Dependencies](#2-module-integration--dependencies)
3. [Code Walkthrough — HTTP/1.1, HTTP/2, HTTP/3 & QUIC Flows](#3-code-walkthrough)
4. [Multiplexing](#4-multiplexing)
5. [Encrypted & Decrypted Flows](#5-encrypted--decrypted-flows)
6. [Adversarial Observations & Risk Areas](#6-adversarial-observations--risk-areas)

---

## 1. Architecture & Design

### 1.1 The Module System

nginx is built from composable modules. Every HTTP module declares itself via `ngx_http_module_t` — a vtable with hooks for config-parse time and run time. The framework calls each hook in a defined order; modules cannot break each other's call chain.

```
ngx_http_module_t {
    preconfiguration       // add variables before config parse
    postconfiguration      // insert phase handlers after config parse
    create_main_conf       // allocate main-level config struct
    init_main_conf         // finalise main-level config
    create_srv_conf        // allocate server-level config struct
    merge_srv_conf         // inherit main → server
    create_loc_conf        // allocate location-level config struct
    merge_loc_conf         // inherit server → location
}
```

There are three configuration contexts (main/http, server, location), each holding a `void **` array indexed by `module->ctx_index`.
See: `src/http/ngx_http.c:123` (`ngx_http_block`), `src/http/ngx_http_config.h`.

### 1.2 Request Processing Phases

The central concept is the **phase engine** — a flat array of `ngx_http_phase_handler_t` structs compiled at config time from all registered handlers. Each handler has a checker function (dispatching logic) and a `next` index (where to jump on skip).

Phases in order (`src/http/ngx_http_core_module.h:121`):

| Phase | Purpose |
|---|---|
| `POST_READ` | Early read (e.g., real IP extraction) |
| `SERVER_REWRITE` | Server-level rewrite rules |
| `FIND_CONFIG` | Location tree lookup (always runs) |
| `REWRITE` | Location-level rewrite rules |
| `POST_REWRITE` | Loop guard — jumps back to FIND_CONFIG if rewritten |
| `PREACCESS` | Rate limiting, connection limits |
| `ACCESS` | Authentication, IP allow/deny |
| `POST_ACCESS` | Handle access deny |
| `PRECONTENT` | `try_files`, mirror |
| `CONTENT` | Response generation (proxy, static files, FastCGI…) |
| `LOG` | Access logging |

Phase handlers are compiled into a linear array by `ngx_http_init_phase_handlers()` (`src/http/ngx_http.c:455`). At runtime `ngx_http_handler()` sets `r->phase_handler = 0` and calls the first checker.

### 1.3 Filter Chains

Two filter chains run after the content phase:

- **Header filter chain**: linked list of `ngx_http_output_header_filter_pt` function pointers. Head: `ngx_http_top_header_filter` (`src/http/ngx_http.c:69`). Each filter transforms `r->headers_out` and calls `ngx_http_next_header_filter`.  
- **Body filter chain**: linked list of `ngx_http_output_body_filter_pt`. Head: `ngx_http_top_body_filter`. Each filter transforms the `ngx_chain_t` buffer list.  
- **Request body filter chain**: `ngx_http_top_request_body_filter` — for transforming inbound body data.

Modules insert themselves into both chains at `postconfiguration` time (`src/http/ngx_http.c:68–71`). Insert order determines chain order; last module to register becomes the first to execute (LIFO stack pattern).

### 1.4 Key Data Structures

| Struct | File | Role |
|---|---|---|
| `ngx_http_request_t` | `src/http/ngx_http_request.h` | Entire request state — all parsed headers, URI, method, phase index, upstream pointer, cache pointer |
| `ngx_http_core_main_conf_t` | `src/http/ngx_http_core_module.h:155` | Global config; holds the phase engine (`phase_engine`), server list, variable hashes |
| `ngx_http_connection_t` | `src/http/ngx_http_request.h` | Per-connection state shared across keep-alive requests; holds `addr_conf`, `ssl`, protocol flags |
| `ngx_http_phase_engine_t` | `src/http/ngx_http_core_module.h:142` | Compiled flat handler array |
| `ngx_http_upstream_t` | `src/http/ngx_http_upstream.h` | Upstream proxy state — peer, buffers, cache, callbacks |

### 1.5 Memory Model

- Every connection gets a pool (`c->pool`) allocated at accept time.
- Every request gets its own pool from that connection pool (`r->pool`).
- H2 gets an extra dedicated pool per `ngx_http_v2_connection_t` (`h2c->pool`) — size controlled by `http2_pool_size`.
- QUIC streams each get a pool from the parent QUIC connection.
- Cleanup handlers attached to pools (`ngx_pool_cleanup_add`) are the primary mechanism for guaranteed resource release.

---

## 2. Module Integration & Dependencies

### 2.1 Startup: How the HTTP Module Initialises

```
main()
  └─ ngx_init_cycle()
       └─ ngx_conf_parse()          # parse nginx.conf
            └─ ngx_http_block()     # triggered by "http {" directive
                 ├─ creates 3 conf contexts (main/srv/loc)
                 ├─ calls create_*_conf() for every http module
                 ├─ ngx_conf_parse() recursively for server{}/location{}
                 ├─ merge_*_conf() bottom-up for each module
                 ├─ ngx_http_init_phase_handlers()   # compile flat phase array
                 └─ ngx_http_optimize_servers()      # build listen socket structures
                       └─ ngx_http_add_listening()
                            └─ ls->handler = ngx_http_init_connection
```

See: `src/http/ngx_http.c:123–336`.

### 2.2 Connection Lifecycle

```
OS accept()  →  ngx_http_init_connection()   (src/http/ngx_http_request.c:~280)
     │
     ├── QUIC port? → ngx_http_v3_init_stream(c)   (src/http/ngx_http_request.c:333)
     │
     ├── SSL port?  → rev->handler = ngx_http_ssl_handshake
     │                  └─ after handshake → ngx_http_ssl_handshake_handler()
     │                        └─ ALPN == "h2"? → ngx_http_v2_init()
     │                        └─ else         → ngx_http_wait_request_handler()
     │
     └── Plaintext → rev->handler = ngx_http_wait_request_handler()
                        └─ H2 preface detected? → ngx_http_v2_init()
                        └─ else                 → ngx_http_create_request()
                                                     → ngx_http_process_request_line()
```

### 2.3 Module Dependencies Graph

```
┌─────────────────────────────────────────────────────┐
│                    nginx core                       │
│   (event loop, memory, log, cycle management)       │
└──────────────────────┬──────────────────────────────┘
                       │  event callbacks
┌──────────────────────▼──────────────────────────────┐
│                  src/http/                          │
│  ngx_http.c         — startup, phase compile        │
│  ngx_http_request.c — connection/request lifecycle  │
│  ngx_http_core_module.c — phases, location tree     │
│  ngx_http_upstream.c    — proxy/cache core          │
│  ngx_http_file_cache.c  — cache storage             │
│  ngx_http_parse.c       — HTTP/1 request parser     │
│  ngx_http_script.c      — variable evaluation       │
└──────┬────────────────┬────────────────┬────────────┘
       │                │                │
  ┌────▼────┐     ┌─────▼────┐    ┌──────▼──────┐
  │ src/http│     │ src/http │    │  src/event/ │
  │   /v2/  │     │   /v3/   │    │    quic/    │
  │ H2 framing   │ H3 framing│   │ QUIC transport
  │ HPACK        │ QPACK     │   │ crypto, ACK  │
  └────┬────┘     └─────┬────┘    └──────┬──────┘
       │                │                │
       └────────────────┴───── SSL/TLS ──┘
                              (src/event/ngx_event_openssl.c)
```

### 2.4 SSL Integration Points

- `ngx_http_request.c:674` — `ngx_http_ssl_handshake()` drives the TLS handshake read loop.
- `ngx_http_request.c:824` — `ngx_http_ssl_handshake_handler()` is the completion callback; reads ALPN to decide H2 vs H1.
- For QUIC, TLS is integrated directly into the QUIC layer (`src/event/quic/ngx_event_quic_ssl.c`), not through the standard `c->ssl` path.

### 2.5 Upstream Module Interface

Content modules (proxy, FastCGI, SCGI, gRPC) do not talk to network directly; they populate `r->upstream` (an `ngx_http_upstream_t`) and call `ngx_http_upstream_init()`. The upstream module manages:

- Peer selection (round-robin, hash, least-conn) via `upstream->peer.init()`.
- Cache check (`ngx_http_upstream_cache()`).
- Connection, header send, response read.
- Filter chain invocation on response.

---

## 3. Code Walkthrough

### 3.1 HTTP/1.1 Flow

```
ngx_http_wait_request_handler(rev)           [ngx_http_request.c:~400]
  └─ ngx_http_create_request(c)              [ngx_http_request.c:538]
       └─ alloc r from c->pool
            └─ r->http_connection = hc
                 └─ r->phase_handler = 0

ngx_http_process_request_line(rev)           [ngx_http_request.c:~900]
  └─ ngx_http_parse_request_line(r, b)       [ngx_http_parse.c]
       └─ → NGX_OK → ngx_http_process_request_headers()
                └─ ngx_http_parse_header_line() in loop
                     └─ → NGX_HTTP_PARSE_HEADER_DONE
                          └─ ngx_http_process_request(r)
                               └─ ngx_http_handler(r)
                                    └─ r->write_event_handler =
                                         ngx_http_core_run_phases
                                    └─ ngx_http_core_run_phases(r)
                                         └─ phase_engine.handlers[r->phase_handler].checker(r, ph)
                                              ... POST_READ → SERVER_REWRITE → FIND_CONFIG
                                              → REWRITE → ACCESS → CONTENT
                                                  └─ content handler (e.g., proxy, static)
                                                       └─ ngx_http_send_header(r)
                                                            └─ ngx_http_top_header_filter(r)
                                                                 └─ ... header filter chain ...
                                                                      └─ ngx_http_write_filter()
                                                       └─ ngx_http_output_filter(r, chain)
                                                            └─ ngx_http_top_body_filter(r, chain)
                                                                 └─ ... body filter chain ...
                                                                      └─ ngx_http_write_filter()
```

### 3.2 HTTP/2 Flow

#### 3.2.1 Connection Initialisation

```
ngx_http_v2_init(rev)                        [src/http/v2/ngx_http_v2.c:204]
  ├─ allocate ngx_http_v2_connection_t (h2c)
  ├─ h2c->send_window = NGX_HTTP_V2_DEFAULT_WINDOW (65535)
  ├─ h2c->recv_window = NGX_HTTP_V2_MAX_WINDOW (2^31-1)
  ├─ h2c->frame_size  = NGX_HTTP_V2_DEFAULT_FRAME_SIZE (16384)
  ├─ h2c->streams_index = hash table (sid → ngx_http_v2_node_t*)
  ├─ ngx_http_v2_send_settings()             # server SETTINGS frame
  ├─ ngx_http_v2_send_window_update(sid=0)   # connection-level window update
  ├─ h2c->state.handler = ngx_http_v2_state_preface   # expect client preface
  └─ rev->handler = ngx_http_v2_read_handler
```

#### 3.2.2 Frame Dispatch State Machine

`ngx_http_v2_read_handler` reads data into a single shared receive buffer (`h2mcf->recv_buffer`), then drives a state machine:

```
State handlers (src/http/v2/ngx_http_v2.c):
  ngx_http_v2_state_preface         → validate "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"
  ngx_http_v2_state_preface_end
  ngx_http_v2_state_head            → parse 9-byte frame header (type, flags, sid, length)
  ngx_http_v2_state_*               → dispatch by frame type:

Frame type → handler array (ngx_http_v2_frame_states):
  0x0 DATA          → ngx_http_v2_state_data
  0x1 HEADERS       → ngx_http_v2_state_headers
                           └─ HPACK decode into r->headers_in
                                └─ ngx_http_v2_state_header_complete()
                                     └─ ngx_http_v2_run_request(r)
                                          └─ ngx_http_handler(r)   ← same phase engine as H1
  0x3 RST_STREAM    → ngx_http_v2_state_rst_stream
  0x4 SETTINGS      → ngx_http_v2_state_settings
  0x6 PING          → ngx_http_v2_state_ping
  0x7 GOAWAY        → ngx_http_v2_state_goaway
  0x8 WINDOW_UPDATE → ngx_http_v2_state_window_update
  0x9 CONTINUATION  → ngx_http_v2_state_continuation
```

#### 3.2.3 Stream Creation

`ngx_http_v2_create_stream()` (`src/http/v2/ngx_http_v2.c:120` declaration) allocates a `ngx_http_v2_stream_t` and a full `ngx_http_request_t`. Each stream is a complete nginx request. The stream is indexed by SID in `h2c->streams_index`.

#### 3.2.4 Response Path

```
ngx_http_v2_header_filter(r)              [src/http/v2/ngx_http_v2_filter_module.c]
  └─ HPACK-encode response headers
       └─ ngx_http_v2_create_headers_frame()
            └─ enqueue to h2c->last_out

ngx_http_v2_send_chain(fc, in, limit)     # fc = fake per-stream connection
  └─ respect stream send_window and connection send_window
       └─ ngx_http_v2_filter_get_data_frame()
            └─ enqueue DATA frames
                 └─ ngx_http_v2_filter_send()
                      └─ ngx_http_v2_send_output_queue(h2c)
                           └─ actual write on real connection c
```

#### 3.2.5 HPACK

HPACK encoding/decoding lives in `src/http/v2/ngx_http_v2_table.c` (static/dynamic header table) and `src/http/v2/ngx_http_v2_encode.c` (encoding primitives). The Huffman codec is shared with H3 QPACK: `src/http/ngx_http_huff_encode.c` / `ngx_http_huff_decode.c`.

### 3.3 HTTP/3 & QUIC Flow

#### 3.3.1 QUIC Engine Entry

HTTP/3 over QUIC starts at the OS level with a **UDP** socket. nginx's listen socket (`type = SOCK_DGRAM`) is set when `quic` flag is present in `ngx_http_listen_opt_t` (`src/http/ngx_http_core_module.h:87`).

```
UDP datagram arrives
  └─ ngx_event_quic_udp.c → ngx_quic_input_handler(rev)   [src/event/quic/ngx_event_quic.c]
       └─ ngx_quic_handle_datagram()
            └─ ngx_quic_handle_packet()    # decrypt, validate connection ID
                 └─ ngx_quic_handle_payload()
                      └─ ngx_quic_handle_frames()
                           └─ per-frame type dispatch:
                                STREAM frame → ngx_quic_handle_stream_frame()
                                                  [src/event/quic/ngx_event_quic_streams.c]
                                CRYPTO frame → feed to TLS via ngx_quic_crypto_provide()
                                ACK frame    → ngx_quic_handle_ack_frame()
                                MAX_DATA     → ngx_quic_handle_max_data_frame()
```

#### 3.3.2 QUIC Stream → HTTP/3 Layer

```
ngx_quic_handle_stream_frame()
  └─ find/create ngx_quic_stream_t in qc->streams rbtree
       └─ stream->connection = new fake ngx_connection_t
            └─ data ready → wake stream's read event
                 └─ ngx_http_v3_init_stream(c)   [src/http/v3/ngx_http_v3_request.c:56]
                      ├─ if c->quic->id & UNIDIRECTIONAL → ngx_http_v3_init_uni_stream()
                      └─ else → ngx_http_v3_init_request_stream()
                                    └─ wait_request_handler for H3
```

#### 3.3.3 HTTP/3 Session Initialisation

```
ngx_http_v3_init(c)                       [src/http/v3/ngx_http_v3_request.c:110]
  ├─ ngx_http_v3_init_session(c)          # allocate ngx_http_v3_session_t
  ├─ start keepalive timer
  ├─ ALPN check: "h3" or "hq-interop"
  ├─ ngx_http_v3_send_settings(c)         # SETTINGS frame on control stream
  └─ if table capacity > 0: open QPACK decoder uni-stream
```

#### 3.3.4 HTTP/3 Request Processing

```
ngx_http_v3_wait_request_handler(rev)     [src/http/v3/ngx_http_v3_request.c]
  └─ read from QUIC stream buf
       └─ ngx_http_v3_process_request(rev)
            └─ ngx_http_v3_parse_headers()    [src/http/v3/ngx_http_v3_parse.c]
                 ├─ parse HEADERS frame (frame type 0x01)
                 │    └─ QPACK decode (ngx_http_v3_parse_field_rep)
                 │         └─ look up static/dynamic table via ngx_http_v3_parse_lookup()
                 └─ → NGX_DONE
                      └─ ngx_http_v3_process_header(r, name, value)  per pseudo-header
                           └─ ngx_http_v3_process_pseudo_header()
                                └─ map :method/:path/:scheme/:authority → r->method, uri, …
                      └─ ngx_http_v3_process_request_header(r)
                           └─ ngx_http_process_request(r)
                                └─ ngx_http_handler(r)   ← same phase engine
```

#### 3.3.5 HTTP/3 Response Path

```
ngx_http_v3_header_filter(r)              [src/http/v3/ngx_http_v3_filter_module.c]
  └─ QPACK-encode headers into buf
       └─ write HEADERS frame (0x01) to QUIC stream

ngx_http_v3_body_filter(r, chain)
  └─ wrap chain buffers into DATA frames (0x00)
       └─ write to QUIC stream
            └─ ngx_quic_stream_send → ngx_quic_output.c → UDP send
```

#### 3.3.6 QPACK

QPACK (HTTP/3 equivalent of HPACK) is implemented in:
- `src/http/v3/ngx_http_v3_table.c` — static table (99 entries) + dynamic table (`ngx_http_v3_dynamic_table_t` in session)
- `src/http/v3/ngx_http_v3_encode.c` + `ngx_http_v3_encode.h` — encoding
- `src/http/v3/ngx_http_v3_parse.c` — decoding
- `src/http/v3/ngx_http_v3_uni.c` + `ngx_http_v3_uni.h` — unidirectional stream management for QPACK encoder/decoder streams

Unidirectional streams by ID convention (`src/http/v3/ngx_http_v3.h:28`):
```
0x00 → Control stream (client)
0x01 → Push stream
0x02 → QPACK Encoder stream
0x03 → QPACK Decoder stream
```

---

## 4. Multiplexing

### 4.1 HTTP/1.1 — No True Multiplexing

HTTP/1.1 is sequential: one request per connection at a time. Pipelining (multiple requests sent without waiting) is supported but responses must be returned in order. nginx tracks pipelining state in `ngx_http_connection_t`.

### 4.2 HTTP/2 Multiplexing

#### Stream identity

Every H2 request has an integer Stream ID (SID), always odd for client-initiated streams. The SID is embedded in every frame's 9-byte header. nginx stores streams in a hash table:

```c
// src/http/v2/ngx_http_v2.h:127
struct ngx_http_v2_connection_s {
    ngx_http_v2_node_t  **streams_index;   // hash table: (sid >> 1) & mask → node
    ngx_uint_t            processing;       // count of live streams
    ...
};
```

Lookup: `ngx_http_v2_get_node_by_id(h2c, sid, alloc)` (`src/http/v2/ngx_http_v2.c`).

#### Flow control — two layers

1. **Connection-level** (`h2c->send_window`, `h2c->recv_window`): total bytes in flight across all streams.
2. **Stream-level** (`stream->send_window`, `stream->recv_window`): per-stream budget.

When a DATA frame arrives, nginx checks both windows:

```c
// src/http/v2/ngx_http_v2.c:~968
if (size > h2c->recv_window) → FLOW_CONTROL_ERROR
h2c->recv_window -= size;
if (h2c->recv_window < NGX_HTTP_V2_MAX_WINDOW / 4)
    → send WINDOW_UPDATE (connection)

// src/http/v2/ngx_http_v2.c:~1003
if (size > stream->recv_window) → FLOW_CONTROL_ERROR
stream->recv_window -= size;
```

On send, `ngx_http_v2_flow_control()` blocks DATA frames if either window is exhausted. Blocked streams queue in `h2c->waiting` (`src/http/v2/ngx_http_v2_filter_module.c:ngx_http_v2_waiting_queue()`).

#### Fake connection per stream

Each H2 stream gets a lightweight `ngx_connection_t` allocated with `h2c->free_fake_connections`. The stream's `c->write->handler = ngx_http_v2_send_chain` allows the rest of nginx to use its standard `send_chain` abstraction without modification. This is the architectural key: every H2 stream looks like a normal nginx connection to the layers above it.

#### Priority (deprecated in RFC 9113 but handled)

`ngx_http_v2_node_t` holds `rank`, `weight`, `rel_weight` and parent/child pointers for the dependency tree. `ngx_http_v2_set_dependency()` and `ngx_http_v2_node_children_update()` maintain it. In practice, priority scheduling is no longer negotiated by modern browsers.

### 4.3 HTTP/3 / QUIC Multiplexing

QUIC provides **stream multiplexing at the transport layer**, solving H2's head-of-line blocking problem. Each QUIC stream is an independent byte-stream with its own sequence numbers and acknowledgements.

#### Stream management

Streams are maintained in a red-black tree keyed by stream ID:

```c
// src/event/quic/ngx_event_quic_connection.h:~160
typedef struct {
    ngx_rbtree_t    tree;       // all streams: find by id
    ngx_rbtree_node_t sentinel;
    ngx_queue_t     uninitialized;
    ngx_queue_t     free;
    uint64_t        server_max_streams_uni;
    uint64_t        server_max_streams_bidi;
    ...
} ngx_quic_streams_t;
```

Lookup: `ngx_quic_find_stream()` (`src/event/quic/ngx_event_quic_streams.c`). New stream: `ngx_quic_handle_stream_frame()` creates a new `ngx_quic_stream_t` and its fake `ngx_connection_t`.

#### Stream ID encoding

QUIC stream IDs encode direction and type in the low 2 bits:
- Bit 0: 0 = client-initiated, 1 = server-initiated  
- Bit 1: 0 = bidirectional, 1 = unidirectional

HTTP/3 request streams are client-initiated bidirectional (IDs 0, 4, 8, …).

#### Flow control — same two-level model

```c
// src/event/quic/ngx_event_quic_connection.h:~155
uint64_t  recv_max_data;      // connection-level receive max
uint64_t  send_max_data;      // connection-level send max
uint64_t  server_max_streams_bidi;  // max concurrent bidi streams
```

Per-stream limits in `ngx_quic_stream_t`. `STREAM_DATA_BLOCKED` / `DATA_BLOCKED` frames carry credit requests. `MAX_STREAM_DATA` / `MAX_DATA` frames grant credit.

#### Congestion control

`ngx_quic_congestion_t` (`src/event/quic/ngx_event_quic_connection.h:~175`) implements CUBIC:
- `in_flight`: unacknowledged bytes
- `window`: congestion window (bytes)
- `ssthresh`: slow-start threshold
- Managed in `src/event/quic/ngx_event_quic_ack.c`

#### QUIC vs H2 HOL comparison

| HOL scenario | H2 (TCP) | H3 (QUIC) |
|---|---|---|
| Packet loss on stream 3 | **Blocks all streams** (TCP stalls) | Stream 3 stalls; others proceed |
| Packet reorder | Stalls all streams | Each stream independent |

---

## 5. Encrypted & Decrypted Flows

### 5.1 HTTP/1.1 TLS (via OpenSSL)

```
ngx_http_init_connection()
  └─ if ssl port: rev->handler = ngx_http_ssl_handshake
                                   [src/http/ngx_http_request.c:674]

ngx_http_ssl_handshake(rev)
  └─ ngx_ssl_handshake(c)               # OpenSSL SSL_accept() wrapper
       └─ if NGX_AGAIN: wait for more data (non-blocking)
       └─ if NGX_OK:
            c->ssl->handler = ngx_http_ssl_handshake_handler

ngx_http_ssl_handshake_handler(c)       [src/http/ngx_http_request.c:824]
  └─ check ALPN:                        # SSL_get0_alpn_selected()
       ├─ "h2" → ngx_http_v2_init(c->read)
       └─ else  → ngx_http_wait_request_handler()   # normal H1 path
```

All subsequent I/O is transparently handled by `c->recv` / `c->send` function pointers which are overridden to OpenSSL wrappers (`c->ssl != NULL`). The HTTP layer never sees raw TLS records.

### 5.2 HTTP/2 TLS

HTTP/2 always runs over TLS (RFC 7540 §3.3 for "https" scheme). The TLS layer is identical to H1 TLS. ALPN negotiation selects "h2" (`NGX_HTTP_V2_ALPN_PROTO = "\x02h2"`, `src/http/v2/ngx_http_v2.h:16`). The H2 layer receives already-decrypted bytes from the TLS layer.

HTTP/2 cleartext (h2c) is also supported — detection happens in `ngx_http_wait_request_handler()` by checking the connection preface `PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n` (`src/http/ngx_http_request.c:501`).

### 5.3 QUIC / HTTP/3 Encryption (Multi-Level)

QUIC has a fundamentally different encryption model. There is no separate TLS connection; TLS 1.3 is **embedded inside QUIC** using a custom record layer API. nginx implements this via callbacks registered with OpenSSL/BoringSSL/quictls.

#### Encryption levels (packet number spaces)

```c
// src/event/quic/ngx_event_quic_connection.h:21
#define NGX_QUIC_ENCRYPTION_INITIAL      0  // AEAD AES-128-GCM, fixed salt
#define NGX_QUIC_ENCRYPTION_EARLY_DATA   1  // 0-RTT (client → server)
#define NGX_QUIC_ENCRYPTION_HANDSHAKE    2  // TLS handshake messages
#define NGX_QUIC_ENCRYPTION_APPLICATION  3  // 1-RTT, all HTTP/3 data
```

Each level has independent packet number spaces with separate cryptographic secrets:

```c
// src/event/quic/ngx_event_quic_protection.h:56
struct ngx_quic_keys_s {
    ngx_quic_secrets_t  secrets[NGX_QUIC_ENCRYPTION_LAST]; // [client/server] per level
    ngx_quic_secrets_t  next_key;   // for key update (QUIC key rotation)
    ngx_uint_t          cipher;
};

typedef struct {
    ngx_quic_secret_t  client;
    ngx_quic_secret_t  server;
} ngx_quic_secrets_t;

typedef struct {
    ngx_quic_md_t      secret;  // HKDF-derived secret
    ngx_quic_iv_t      iv;      // 12-byte nonce base
    ngx_quic_md_t      hp;      // header protection key
    ngx_quic_crypto_ctx_t *ctx; // AEAD context
    EVP_CIPHER_CTX    *hp_ctx;  // header protection context
} ngx_quic_secret_t;
```

#### QUIC handshake sequence (TLS 1.3 inside QUIC)

```
Client → Server: Initial packet (CRYPTO frame, ClientHello)
   Level: INITIAL, key = HKDF(QUIC_INITIAL_SALT, DCID)

Server:
  ngx_quic_handle_payload() → CRYPTO frame at INITIAL level
  ngx_quic_crypto_provide() → SSL_provide_quic_data() (BoringSSL/quictls API)
                                   or quic CBS callbacks (OpenSSL QUIC API)
  TLS processes ClientHello → yields secrets:
    ngx_quic_keys_set_encryption_secret(level=HANDSHAKE, ...)
    ngx_quic_keys_set_encryption_secret(level=APPLICATION, ...)

Server → Client: Initial (ACK) + Handshake packets
  Level: HANDSHAKE (ServerHello, EncryptedExtensions, Certificate, Verify, Finished)

Client → Server: Handshake Finished
  Both derive APPLICATION-level keys.

Post-handshake: all HTTP/3 data in APPLICATION (1-RTT) packets.
```

TLS secret delivery callback paths:

- **BoringSSL/AWS-LC**: `ngx_quic_set_read_secret()` + `ngx_quic_set_write_secret()` (`src/event/quic/ngx_event_quic_ssl.c`)
- **quictls**: `ngx_quic_set_encryption_secrets()`
- **OpenSSL QUIC API**: `ngx_quic_cbs_yield_secret()` (`src/event/quic/ngx_event_quic_ssl.c:~26`)

All paths ultimately call `ngx_quic_keys_set_encryption_secret()` (`src/event/quic/ngx_event_quic_protection.c`) to derive per-level keys via HKDF.

#### QUIC packet encryption/decryption mechanics

Every QUIC packet has two protection layers:

1. **Header protection** — the packet number and some header flags are XOR-masked using a per-level header-protection key (`hp_ctx`). This prevents on-path observers from reading packet numbers.
2. **Payload encryption** — AEAD (AES-128-GCM or ChaCha20-Poly1305) with a per-packet nonce derived as `IV XOR packet_number`.

```
Decrypt path (ngx_event_quic_protection.c):
  ngx_quic_decrypt()
    └─ sample 16 bytes from encrypted payload
         └─ AES-ECB(hp_key, sample) → mask
              └─ unmask packet number field (xor)
                   └─ reconstruct full 64-bit pnum from short encoding
                        └─ compute nonce = iv XOR pnum
                             └─ AEAD_Open(ctx, nonce, aad=header, ciphertext) → plaintext

Encrypt path:
  ngx_quic_encrypt()
    └─ AEAD_Seal() → ciphertext
         └─ compute mask from sample
              └─ mask packet number and header bits
```

#### Key update (post-handshake)

RFC 9001 §6 requires support for key rotation. nginx handles this:
- Detects key phase bit flip in a received packet → switches to `qc->keys->next_key`.
- `src/event/quic/ngx_event_quic_connection.h`: `key_phase:1` and `key_update` event.
- `ngx_quic_keys_s.next_key` holds the next secrets during transition.

#### 0-RTT (Early Data)

`NGX_QUIC_ENCRYPTION_EARLY_DATA` (level 1) allows clients to send HTTP/3 requests before the handshake completes, using a pre-shared session ticket. nginx can receive 0-RTT packets but anti-replay protection is the application's responsibility (documented risk: 0-RTT is inherently replayable without external state).

### 5.4 Encrypt/Decrypt Boundary Diagram

```
HTTP/1.1 over TLS:
  [TCP bytes] → OpenSSL decrypt → [plaintext HTTP] → nginx request pipeline

HTTP/2 over TLS:
  [TCP bytes] → OpenSSL decrypt → [H2 frames] → frame dispatch → [per-stream request pipeline]

HTTP/3 over QUIC:
  [UDP datagrams]
       └─ header unprotect (AES-ECB mask)
            └─ AEAD decrypt (per encryption level)
                 └─ QUIC frames
                      └─ STREAM frames → QUIC stream buffers
                           └─ [HTTP/3 frames: HEADERS, DATA]
                                └─ QPACK decode
                                     └─ nginx request pipeline
```

### 5.5 No Decryption in Non-TLS Mode

On `http://` addresses without QUIC, nginx reads raw TCP bytes. `c->recv` points to `ngx_unix_recv` (or epoll equiv); no decryption layer. All H2 cleartext detection is byte comparison of the preface string.

---

## 6. Adversarial Observations & Risk Areas

### 6.1 Shared H2 Receive Buffer

The entire H2 connection uses one receive buffer (`h2mcf->recv_buffer`, `src/http/v2/ngx_http_v2.c:226`). If a slow stream holds the buffer, all other streams on that connection cannot read. Ensure `http2_recv_buffer_size` is sized for peak concurrency.

**Validation**: instrument `h2c->processing` counter under load; alert if consistently near `http2_max_concurrent_streams`.

### 6.2 Stream SID Hash Collision

`ngx_http_v2_index()` uses `(sid >> 1) & mask`. With default `streams_index_mask`, probe chains can degrade O(1) lookup to O(n). A malicious client sending many streams with colliding hash indices causes CPU amplification.

**Validation**: fuzz the SID space; measure lookup time under adversarial SID selection.

### 6.3 QUIC Connection ID Tracking

QUIC connection migration (`src/event/quic/ngx_event_quic_migration.c`) requires mapping new client addresses to existing connection state. Incorrect validation of PATH_CHALLENGE / PATH_RESPONSE could allow source-address spoofing amplification.

**Validation**: send forged PATH_RESPONSE on an alternative path without the challenge; verify nginx rejects it.

### 6.4 QPACK Dynamic Table OOM

`ngx_http_v3_dynamic_table_t` in the session (`src/http/v3/ngx_http_v3.h:139`) grows as the encoder inserts entries. If `max_table_capacity` is large and the encoder fills it aggressively, nginx must evict old entries or return errors. Verify eviction does not use stale pointers.

**Validation**: send QPACK encoder stream entries up to `max_table_capacity`; verify memory stays bounded.

### 6.5 0-RTT Replay Window

Early data at `NGX_QUIC_ENCRYPTION_EARLY_DATA` is replayed by proxies or adversaries during reconnect. nginx accepts 0-RTT without application-level deduplication. Any non-idempotent operation (POST, state mutation via proxy) is at risk.

**Mitigation**: disable 0-RTT for endpoints serving non-idempotent operations; use `quic_retry on` (forces stateless Retry, which invalidates old 0-RTT keys).

### 6.6 Header Compression Bomb (HPACK/QPACK)

Both H2 HPACK and H3 QPACK allow dynamic table insertion. A client can craft headers that decompress to very large values. nginx limits this via `large_client_header_buffers` (for H2, checked in `h2c->state.header_limit`; for H3, `st->header_limit` in `ngx_http_v3_parse.c`).

**Validation**: send a HEADERS frame referencing many dynamic table entries that sum to > header limit; verify nginx rejects with appropriate error frame.

### 6.7 Phase Handler Ordering Assumptions

Modules that call `ngx_http_add_module_header_filter()` at postconfiguration time insert themselves into a LIFO stack. If two modules assume they are adjacent in the chain, load order in `nginx.conf` can silently break both. There is no ordering guarantee beyond "last to register runs first."

**Mitigation**: document assumed chain positions for any module that depends on upstream module state in the filter chain.

---

## Quick Reference: File Index

| File | Purpose |
|---|---|
| `src/http/ngx_http.c` | Bootstrap: `ngx_http_block`, phase compile, listen setup |
| `src/http/ngx_http_request.c` | Connection init, SSL, H2/H3 dispatch, request lifecycle |
| `src/http/ngx_http_core_module.c` | Phase checkers, location tree, `ngx_http_handler` |
| `src/http/ngx_http_parse.c` | HTTP/1.1 request-line and header parser |
| `src/http/ngx_http_upstream.c` | Proxy/cache upstream framework |
| `src/http/ngx_http_file_cache.c` | Disk cache: lookup, store, key hashing (MD5+CRC32) |
| `src/http/v2/ngx_http_v2.c` | H2 connection init, frame state machine, flow control |
| `src/http/v2/ngx_http_v2_filter_module.c` | H2 header/body filters, DATA frame assembly |
| `src/http/v2/ngx_http_v2_table.c` | HPACK static + dynamic header table |
| `src/http/v2/ngx_http_v2_encode.c` | HPACK integer/string encoding |
| `src/http/v2/ngx_http_v2_module.c` | H2 directives (`http2`, `http2_max_concurrent_streams`, …) |
| `src/http/v3/ngx_http_v3_request.c` | H3 stream init, request processing, body read |
| `src/http/v3/ngx_http_v3_parse.c` | QPACK decode, varint parse, frame type dispatch |
| `src/http/v3/ngx_http_v3_filter_module.c` | H3 header/body filters, QPACK encode |
| `src/http/v3/ngx_http_v3_table.c` | QPACK static (99 entries) + dynamic table |
| `src/http/v3/ngx_http_v3_uni.c` | Unidirectional stream management (control, encoder, decoder) |
| `src/http/v3/ngx_http_v3_module.c` | H3/QUIC directives (`http3`, `quic_retry`, `quic_gso`, …) |
| `src/event/quic/ngx_event_quic.c` | QUIC entry: datagram handler, packet dispatch |
| `src/event/quic/ngx_event_quic_protection.c` | AEAD encrypt/decrypt, header protection, key derivation |
| `src/event/quic/ngx_event_quic_ssl.c` | TLS integration callbacks (secret delivery, handshake) |
| `src/event/quic/ngx_event_quic_streams.c` | QUIC stream rbtree, STREAM/MAX_DATA frame handlers |
| `src/event/quic/ngx_event_quic_ack.c` | QUIC ACK processing, PTO, congestion control |
| `src/event/quic/ngx_event_quic_output.c` | Packet assembly, coalescing, UDP send |
| `src/event/quic/ngx_event_quic_connection.h` | Master struct definitions: `ngx_quic_connection_t`, `ngx_quic_keys_s`, send contexts |
| `src/http/ngx_http_huff_encode.c` | Shared Huffman codec (used by both H2 HPACK and H3 QPACK) |
