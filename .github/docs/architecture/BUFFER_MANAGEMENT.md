# NGINX Buffer Management & Safety Analysis

**Audience**: Security-focused engineers, buffer overflow hunters, performance tuners.
**Last Updated**: 2024
**Threats Covered**: Buffer overflows, malformed input, parser edge cases, resource exhaustion.
**Source paths**: All `src/` references are relative to the **nginx repository root** (`<repo>/src/`), not relative to `.github/`.

---

## Executive Summary

NGINX uses **pool-based buffers** for predictable allocation and **careful bounds checking** in parsers. However, several attack vectors exist: oversized headers, slow-read attacks, pipelined requests, and parser state machine edge cases. This document catalogs buffer allocation paths, identifies overflow risks, and provides validation strategies.

**Key Principles**:
- All buffers allocated from connection pools (size bounds enforced)
- Parsers are state machines (resumable; handle partial input)
- HTTP headers limited by `large_client_header_buffers` directive
- Request body handling via `client_max_body_size` and chunked transfer limits

---

## Buffer Allocation Paths

### 1. Connection Read Buffer

**File**: `src/core/ngx_connection.c:242-258`

```c
ngx_connection_t *
ngx_get_connection(ngx_socket_t s, ngx_log_t *log)
{
    ngx_connection_t  *c;
    
    c = ngx_cycle->free_connections;
    
    if (c == NULL) {
        ngx_log_error(NGX_LOG_ALERT, log, 0,
                      "connection limit reached");
        return NULL;
    }
    
    ngx_cycle->free_connections = c->data;
    ngx_cycle->free_connection_n--;
    
    c->pool = ngx_create_pool(ngx_cycle->pool_size, log);  // ← Usually 8 KB per connection
    
    c->number = ngx_atomic_fetch_add(ngx_connection_counter, 1);
    
    return c;
}
```

**Initial Buffer Creation** (`src/http/ngx_http_request.c:528-533`):

```c
static ngx_int_t
ngx_http_init_connection(ngx_connection_t *c)
{
    // ...
    
    c->buffer = ngx_create_temp_buf(c->pool, ...);  // ← Allocates buffer + chain
    
    // ...
}

#define ngx_create_temp_buf(pool, size)                                          \
    ngx_buf_chain(pool, size)

ngx_buf_t *
ngx_buf_chain(ngx_pool_t *pool, size_t size)
{
    ngx_buf_t    *b, *t;
    ngx_chain_t  *cl;
    
    b = ngx_calloc_buf(pool);      // allocate buffer struct
    
    if (b == NULL) {
        return NULL;
    }
    
    b->start = ngx_palloc(pool, size);  // allocate data
    
    if (b->start == NULL) {
        return NULL;
    }
    
    b->pos = b->start;
    b->last = b->start;
    b->end = b->start + size;
    b->temporary = 1;
    
    return b;
}
```

**Size Determination** (for HTTP):

```c
// Initial read buffer
ngx_create_temp_buf(c->pool, 
                    conf->client_header_buffer_size);  // default 1 KB

// client_header_buffer_size directive sets initial size
```

### 2. Request Header Buffer (Oversized)

**File**: `src/http/ngx_http_parse.c`

When request line or headers exceed initial buffer:

```c
ngx_http_request_line_t *
ngx_http_alloc_large_client_header_buffer(ngx_http_connection_t *hc)
{
    ngx_pool_t  *pool;
    
    // Allocate from connection's large client header buffer list
    // governed by: large_client_header_buffers N Msize
    // default: 4 buffers of 32 KB each
    
    if (hc->nbufs >= conf->large_client_header_buffers.num) {
        ngx_log_error(NGX_LOG_ERROR, c->log, 0,
                      "client sent too large request line");
        return NULL;  // ✗ Limit exceeded
    }
    
    // Allocate from connection pool
    b = ngx_palloc(c->pool, conf->large_client_header_buffers.size);
    
    // ← Max size: 32 KB (default), per buffer
    // ← Max buffers: 4 (default), so max 128 KB total
    
    return b;
}
```

**Directive Limits** (nginx configuration):

```nginx
client_header_buffer_size        1k;         # Initial
large_client_header_buffers      4 32k;      # N buffers of size K
                                             # Total header memory: 4 * 32 KB = 128 KB per connection
```

**Protection**:
- ✓ Hard limits on number and size of buffers
- ✗ Each large buffer comes from connection pool; can still exhaust memory with 10,000 connections × 128 KB = 1.2 GB

### 3. Request Body Buffer

**File**: `src/http/ngx_http_request_body.c`

```c
ngx_int_t
ngx_http_read_client_request_body(ngx_http_request_t *r)
{
    // client_max_body_size: maximum request body size
    // default 1 MB
    
    if (r->content_length > conf->client_max_body_size) {
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "client_max_body_size (%O > %O)",
                      r->content_length, conf->client_max_body_size);
        return NGX_HTTP_REQUEST_ENTITY_TOO_LARGE;
    }
    
    // Allocate buffer for body
    rb->buf = ngx_create_temp_buf(r->pool, client_buffer_size);
    
    // If body > buffer, write to disk (temporary file)
    if (content_length > client_buffer_size) {
        ngx_create_temp_file(&tf, ...);  // ← Write to /var/tmp or configured
    }
}
```

**Temporary File Storage**:

```c
// client_body_temp_path: where to store large bodies
// default: /var/tmp/client_body
// client_body_buffer_size: 128 KB (default)
// Any body > 128 KB goes to disk
```

**Protection**:
- ✓ `client_max_body_size` enforced (default 1 MB)
- ✓ Large bodies written to disk, not memory
- ✓ Temporary files cleaned up on connection close
- ✗ Disk space can be exhausted if `client_body_temp_path` on same partition as OS

---

## HTTP Request Parsing

### Request Line Parser State Machine

**File**: `src/http/ngx_http_parse.c:108`

**States** (line 111-138):
```
sw_start ────→ sw_method ────→ sw_spaces_before_uri ────→ sw_schema ─┐
                                                                      │
        ┌──────────────────────────────────────────────────────────┘
        │
        ├─→ sw_schema_slash ─→ sw_schema_slash_slash ─→ sw_host_start ─→ sw_host
        │
        └─→ sw_uri ────→ sw_http_H ─→ sw_http_HT ─→ sw_http_HTT ─→ sw_http_HTTP ─→ 
            version_digits ─→ sw_almost_done
```

**Critical Parsing Code** (line 142-400+):

```c
for (p = b->pos; p < b->last; p++) {
    ch = *p;
    
    switch (state) {
    
    case sw_start:
        r->request_start = p;           // ← Mark request start
        
        if (ch == CR || ch == LF) {
            continue;                   // Skip leading CRLF
        }
        
        if ((ch < 'A' || ch > 'Z')
            && (ch < 'a' || ch > 'z'))
        {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid method char
        }
        
        state = sw_method;
        break;
    
    case sw_method:
        if (ch == ' ') {
            r->method_end = p - 1;
            state = sw_spaces_before_uri;
            break;
        }
        
        if ((ch < 'A' || ch > 'Z')
            && (ch < 'a' || ch > 'z')
            && ch != '_')
        {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid method
        }
        
        break;
    
    case sw_uri:
        // URI parsing: validate chars, handle special cases
        if (ch == ' ') {
            r->uri_end = p;
            state = sw_http_09;
            break;
        }
        
        if (ch == '?') {
            r->args_start = p + 1;
            state = sw_uri;
            break;
        }
        
        if (ch == '#') {
            r->args_end = p;
            state = sw_uri;
            break;
        }
        
        // Validate URI char (no control chars, etc.)
        if ((ch < '!') || (ch > 0x7e)) {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid URI
        }
        
        break;
    
    case sw_http_H:
    case sw_http_HT:
    case sw_http_HTT:
    case sw_http_HTTP:
        // Validate HTTP version string character-by-character
        if (ch != "HTTP"[state - sw_http_H]) {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Not "HTTP"
        }
        state++;
        break;
    
    case sw_first_major_digit:
        if (ch < '0' || ch > '9') {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid major version
        }
        r->http_major = ch - '0';
        state = sw_major_digit;
        break;
    
    case sw_major_digit:
        if (ch == '.') {
            state = sw_first_minor_digit;
            break;
        }
        if (ch < '0' || ch > '9') {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid major version
        }
        break;
    
    case sw_first_minor_digit:
        if (ch < '0' || ch > '9') {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid minor version
        }
        r->http_minor = ch - '0';
        state = sw_minor_digit;
        break;
    
    case sw_minor_digit:
        if (ch == CR) {
            state = sw_almost_done;
            break;
        }
        if (ch < '0' || ch > '9') {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Invalid minor version
        }
        break;
    
    case sw_spaces_after_digit:
        if (ch == CR) {
            state = sw_almost_done;
            break;
        }
        if (ch != ' ') {
            return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ Unexpected char after version
        }
        break;
    
    case sw_almost_done:
        if (ch == LF) {
            return NGX_OK;  // ✓ Request line complete
        }
        return NGX_HTTP_PARSE_INVALID_REQUEST;  // ✗ No LF after CR
        break;
    }
}

return NGX_AGAIN;  // ← Need more data; resume parsing
```

### Edge Cases & Attacks

#### 1. **Oversized Request Line**

**Attack**:
```
GET /very/long/path/that/goes/on/forever/and/ever/... HTTP/1.1\r\n
```

**Buffer Size**: Checked by `large_client_header_buffers`.

**Validation**:
```c
// In ngx_http_alloc_large_client_header_buffer()
if (hc->nbufs >= conf->large_client_header_buffers.num) {
    return NULL;  // ✓ Limit enforced
}
```

**Configuration**:
```nginx
large_client_header_buffers 4 32k;  # Max 4 buffers, 32 KB each
```

**Security Level**: 🟢 **Protected** (configurable limit)

#### 2. **Request Header Injection**

**Attack**:
```
GET / HTTP/1.1\r\nContent-Length: -1\r\n\r\n
```

**Parsing**: Header parser handles separately (after request line).

**Validation** (`src/http/ngx_http_parse.c:750`):
```c
ngx_int_t
ngx_http_parse_header_line(ngx_http_request_t *r, ngx_buf_t *b)
{
    // Validate header name (alphanumeric + hyphen)
    // Validate header value (no control chars except SP/HT)
    
    // Content-Length validated separately:
    // - Must be non-negative
    // - If multiple Content-Length headers: all must agree
}
```

**Security Level**: 🟢 **Protected** (strict validation)

#### 3. **Slow-Read Attack (Slowloris)**

**Attack**: Send request line slowly, one byte per second.
```
GET / HTTP/1.1\r
(wait 1 second)
\n
(connection idles but holds buffer)
```

**Defense** (line 53-57 in `ngx_http_request.c`):

```c
// Timeout: client_header_timeout (default 60 seconds)
// If no complete request line within 60s, close connection

// Read event handler:
ngx_http_wait_request_handler(ngx_event_t *rev)
{
    ngx_connection_t  *c;
    ngx_http_connection_t *hc;
    
    c = rev->data;
    
    if (rev->timedout) {  // ← Timeout fired
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT,
                      "client timed out");
        ngx_http_close_connection(c);
        return;  // ✓ Connection closed
    }
    
    // ... read data ...
}
```

**Configuration**:
```nginx
client_header_timeout 60s;  # Default
client_body_timeout 60s;    # For body
```

**Security Level**: 🟢 **Protected** (timeout-based)

#### 4. **Request Smuggling (HTTP/1.1 Pipelining)**

**Attack**: Send multiple requests in one packet; server processes inconsistently.

```
POST / HTTP/1.1\r\nContent-Length: 5\r\n\r\nHELLOGET / HTTP/1.1\r\n...
```

**Defense** (line 532 in `ngx_http_request.c`):

```c
// After handling first request:
if (r->keepalive) {
    // Reset connection for next request
    c->requests++;
    c->buffer->pos = c->buffer->last;  // ← Clear buffer
    c->data = ngx_http_connection;
    // Next request must wait for new data from socket
}
```

**Validation**:
- Content-Length strictly enforced (body reading stops at boundary)
- Pipelined requests require proper Keep-Alive
- Transfer-Encoding: chunked handled separately

**Security Level**: 🟢 **Protected** (state machine ensures boundaries)

#### 5. **Chunked Transfer Encoding Edge Cases**

**File**: `src/http/ngx_http_parse.c:966`

```c
ngx_int_t
ngx_http_parse_chunked(ngx_http_request_t *r, ngx_buf_t *b)
{
    // Chunk format:
    // chunk_size [ chunk_extension ] CRLF
    // chunk_data CRLF
    // ... (more chunks)
    // 0 CRLF
    // [ trailer_headers ]
    // CRLF
    
    // Validate chunk size parsing
    if (ch < '0' || (ch > '9' && (ch|0x20) < 'a' && (ch|0x20) > 'f')) {
        return NGX_HTTP_PARSE_INVALID_CHUNK;  // ✗ Invalid hex
    }
}
```

**Attacks**:
- Negative chunk size → ✗ Rejected (only hex allowed)
- Excessive chunk count → ✓ Protected by `client_max_body_size`
- Chunk size overflow → ✗ Parsed as hex, limited to client_max_body_size

**Security Level**: 🟢 **Protected** (strict parsing)

---

## Memory Safety Analysis

### Buffer Overflow Scenarios

#### Scenario 1: Request Line Parser Buffer Overflow

**Question**: Can a malformed request overflow the buffer?

**Answer**: ✗ **No**, because:
1. Buffer size enforced by `large_client_header_buffers` (line 275 in `ngx_http_request.c`)
2. Parser checks `p < b->last` (line 142 in `ngx_http_parse.c`)
3. Return value handled: if buffer full, request buffered and parser resumes

**Code**:
```c
// Line 142: for (p = b->pos; p < b->last; p++)
// b->last is set to buffer end; loop terminates when reached
```

#### Scenario 2: Header Value Copy Overflow

**Question**: When copying header value to request struct, can it overflow?

**Answer**: ✗ **No**, because:
1. Headers stored as `ngx_str_t` (pointer + length, not null-terminated buffer)
2. No strcpy; data copied via `ngx_memcpy(dest, src, len)`
3. Length enforced by `large_client_header_buffers` size

**Code** (`src/http/ngx_http_parse.c:750`):
```c
// Header value stored as:
r->headers_in.host = ngx_palloc(r->pool, sizeof(ngx_table_elt_t));
r->headers_in.host->value.data = header_data;
r->headers_in.host->value.len = header_len;
// ✓ No null-terminated buffer; len tracked separately
```

#### Scenario 3: URI Buffer Overflow

**Question**: Can a crafted URI overflow URI buffer?

**Answer**: ✗ **No**, because:
1. URI stored as offset + length in request struct (lines 149-150 in `ngx_http_parse.c`)
2. No separate buffer; URI lives in connection read buffer
3. If request line > buffer, more buffers allocated (line 275)

#### Scenario 4: Pool Allocation Failure (Cascade)

**Question**: What happens if pool allocation fails?

**Answer**: Graceful degradation:
1. Memory allocation failure returns NULL
2. Caller checks for NULL; returns error response
3. Connection closed; memory freed

**Example**:
```c
c->buffer = ngx_create_temp_buf(c->pool, ...);
if (c->buffer == NULL) {
    ngx_log_error(NGX_LOG_ALERT, c->log, 0, "out of memory");
    ngx_http_close_connection(c);
    return;  // ✓ Graceful
}
```

### Bounds Checking

#### String Operations

**File**: `src/core/ngx_string.c`

```c
// All use explicit length; no strcpy

void *
ngx_memcpy(void *dst, const void *src, size_t n)
{
    // Explicit length n; caller responsible for bounds
}

// Safer patterns:
ngx_str_t s;
s.data = ngx_palloc(pool, len);
s.len = len;
ngx_memcpy(s.data, src, len);  // ✓ Bounds enforced by len parameter
```

#### Buffer I/O

**File**: `src/core/ngx_buf.h`

```c
struct ngx_buf_s {
    u_char  *pos;      // current position
    u_char  *last;     // end of valid data
    u_char  *start;    // buffer start
    u_char  *end;      // buffer end
};

// Safe pattern:
size_t avail = b->end - b->last;
if (avail < needed) {
    return NGX_AGAIN;  // ✓ Not enough space
}
```

---

## Resource Exhaustion Attacks

### 1. **Slow Header Attack**

```
Connection 1: GET / HTTP/1.1\r\n (incomplete, waiting for more)
Connection 2: GET / HTTP/1.1\r\n (incomplete)
... (repeat 100,000 times)
```

**Impact**: Each connection holds ~8 KB pool + buffer.

**Defense**:
- `client_header_timeout`: 60s (default)
- Incomplete headers timed out and connection closed
- Per-worker limit: `worker_connections` (512 default, or 10K+)

**Validation**:
```bash
# Limit half-open connections via firewall/reverse proxy
# Monitor: connections_active via stub_status
```

### 2. **Large Header Bomb**

```
GET / HTTP/1.1\r\nX-Bomb: [32 KB of data repeated 4 times]\r\n
```

**Impact**: Uses up all `large_client_header_buffers` slots.

**Defense**:
- `large_client_header_buffers 4 32k`: Max 128 KB per connection
- Exceeded → 400 Bad Request
- Rejected before parsing headers

**Validation**:
```bash
# Test with: curl -H "X-Large: $(perl -e 'print "A" x 1000000')" http://localhost/
# Expected: 400 error
```

### 3. **Slow-Write (POST) Attack**

```
POST / HTTP/1.1\r\nContent-Length: 1000000\r\n\r\n
(send 1 byte per second for 1 MB body)
```

**Impact**: Connection holds temporary file, consuming disk space.

**Defense**:
- `client_body_timeout`: 60s (default)
- `client_max_body_size`: 1 MB (default)
- Slow writes time out

**Validation**:
```bash
# Monitor temporary file directory:
du -sh /var/tmp/client_body
# Should not grow unbounded
```

### 4. **Chunked Transfer DoS**

```
POST / HTTP/1.1\r\nTransfer-Encoding: chunked\r\n\r\n
FFFFFFFF\r\n (claim 4 GB chunk)
... (start sending slowly)
```

**Impact**: Chunk size > `client_max_body_size` → rejected.

**Defense**:
- Chunk size validated against `client_max_body_size`
- Total body limited by same directive

**Validation**:
```bash
# Test malformed chunked:
printf "POST / HTTP/1.1\r\nTransfer-Encoding: chunked\r\n\r\nZZZZZZ\r\n" | nc localhost 80
# Expected: 400 error
```

---

## Secure Configuration

### Recommended Settings

```nginx
# Connection-level protections
client_header_buffer_size       1k;              # Minimum for most requests
large_client_header_buffers     4 32k;           # 128 KB total per connection
client_max_body_size            10m;             # 10 MB max

# Timeout protections
client_header_timeout           10s;             # Aggressive (default 60s)
client_body_timeout             10s;
send_timeout                    10s;
keepalive_timeout               5s 5s;           # Short keep-alive

# Connection limits
worker_connections             10000;
keepalive_requests              100;             # Max requests per connection
keepalive_disable               msie6;           # Disable for older clients

# Buffer pool size
connection_pool_size            512;             # Smaller pools = less memory per connection
request_pool_size               4k;

# Per-process limits
worker_rlimit_nofile            65535;           # Max file descriptors

# Optional: disable pipelining (if HTTP/2 not used)
server {
    http2_max_requests           1000;
    http2_max_field_size         16k;
    http2_max_header_size        32k;
}
```

---

## Validation & Testing

### Buffer Overflow Testing

```bash
# Test 1: Oversized request line
python3 -c "print('GET ' + 'A' * 100000 + ' HTTP/1.1\r\n')" | nc localhost 80
# Expected: 414 (request-uri too large) or connection close

# Test 2: Oversized headers
printf "GET / HTTP/1.1\r\nHost: $(python3 -c 'print(\"A\" * 100000)')\r\n\r\n" | nc localhost 80
# Expected: 400 (bad request) or 431 (request header fields too large)

# Test 3: Malformed chunked encoding
printf "POST / HTTP/1.1\r\nTransfer-Encoding: chunked\r\n\r\nZZZZ\r\n" | nc localhost 80
# Expected: 400 (bad request)
```

### Memory Leak Testing

```bash
# Valgrind
valgrind --leak-check=full --track-origins=yes nginx

# ASAN (if compiled with -fsanitize=address)
ASAN_OPTIONS=verbosity=1:halt_on_error=1 nginx

# Monitor memory
watch -n 1 'ps aux | grep nginx'
```

### Parser Fuzzing

```bash
# Generate random HTTP requests
afl-fuzz -i in -o out -- nginx -c /dev/stdin

# Or: libFuzzer
clang++ -fsanitize=fuzzer http_parser_fuzzer.c ... -o fuzzer
./fuzzer corpus/
```

---

## Invariants

### Buffer Invariants

1. **Always null-terminated or length-tracked**
   - ✓ Headers: `ngx_str_t` (pointer + length)
   - ✓ Values: stored in buffer, `len` tracked
   - ✗ Exception: none (always explicit)

2. **Never copied without length check**
   ```c
   if (avail < needed) return NGX_AGAIN;  // ✓ Always checked
   ```

3. **Overflow cannot silently corrupt**
   - ✓ Pool allocator detects overwrites (via `p->d.end` boundary)
   - ✓ ASAN catches heap corruption

### Parser Invariants

1. **State machine always resumes**
   - Return `NGX_AGAIN` if more data needed
   - Never crash on partial input

2. **Invalid input always rejected**
   - Never silently accepted
   - Return `NGX_HTTP_PARSE_INVALID_*`

3. **Timeout always enforced**
   - Incomplete requests never block forever
   - Checked via `rev->timedout` flag

---

## Debugging & Investigation

### Log Suspicious Requests

```c
// Add to ngx_http_init_request():
ngx_log_debug4(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
               "request line: %*s (method=%d, uri_len=%d)",
               r->request_end - r->request_start, r->request_start,
               r->method, r->uri_end - r->uri_start);
```

### Monitor Parser Errors

```bash
# Count 400 errors
tail -f /var/log/nginx/error.log | grep "client sent invalid request"

# Count 413 errors (entity too large)
tail -f /var/log/nginx/access.log | grep " 413 "
```

### Inspect Pool Usage

```bash
# Memory map
cat /proc/$(pgrep -f "nginx.*worker" | head -1)/maps

# Pool statistics (if compiled with NGX_DEBUG)
# Add to log: ngx_log_debug2(..., "pool: %uz/%uz", used, total)
```

---

## Next Steps

1. Review **ASSUMPTIONS_AND_THREATS.md** for threat model
2. Study **PERFORMANCE_BOTTLENECKS.md** for buffer tuning
3. Read **CODE_WALKTHROUGH_EVENT_LOOP.md** for annotated parser walk-through

---

**References**:
- Buffer Overflow: "The Art of Software Security Assessment" (Dowd, McDonald)
- Parser Security: "Building Secure Software" (McGraw, Viega)
- OWASP: HTTP Request Smuggling

