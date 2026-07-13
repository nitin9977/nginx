# RFC 9112 — HTTP/1.1 (STD 99)

> Fielding, Nottingham, Reschke, Eds. — June 2022. Obsoletes 7230 (framing
> portions). Updated by RFC 9931 (CONNECT/Upgrade security).
> Full text: <https://www.rfc-editor.org/rfc/rfc9112.txt>

The **wire format** for HTTP/1.1: how a message is serialized as octets on a
connection, how its length is determined, and how connections are managed.
This is the highest-risk area for security bugs (request smuggling). Maps
directly to `src/http/ngx_http_parse.c` and `ngx_http_request.c`.

## Section Map

| § | Topic |
|---|-------|
| 2 | Message format — `start-line`, header section, empty line, message body |
| 3 | Request line — `method SP request-target SP HTTP-version CRLF` |
| 4 | Status line — `HTTP-version SP status-code SP [reason] CRLF` |
| 5 | Field syntax & line folding (obs-fold) |
| 6 | Message body length — the framing algorithm (critical) |
| 7 | Transfer codings — chunked, `Transfer-Encoding`, `TE`, trailers |
| 8 | Handling incomplete messages |
| 9 | Connection management — persistence, `Connection`, pipelining, `Upgrade` |
| 10 | Enclosing messages as data (`message/http`) |
| 11 | Security considerations — **§11.2 request smuggling** |

## Message Grammar (§2–§5)

```
HTTP-message  = start-line CRLF
                *( field-line CRLF )
                CRLF
                [ message-body ]
start-line    = request-line / status-line
request-line  = method SP request-target SP HTTP-version
status-line   = HTTP-version SP status-code SP [ reason-phrase ]
HTTP-version  = "HTTP/" DIGIT "." DIGIT
```

- Only a single `SP` between tokens; recipients SHOULD be strict.
- `request-target` forms: **origin-form** (`/path?query`), **absolute-form**
  (proxy requests), **authority-form** (`CONNECT` only), **asterisk-form**
  (`OPTIONS *`).
- A recipient that receives whitespace between the start-line and the first
  header field, or bare CR, must handle it defensively (see smuggling below).

## Message Body Length (§6.3) — the framing algorithm

Determine body length in this order:

1. Responses to `HEAD`, `1xx`, `204`, `304`, and `2xx` to `CONNECT` have **no
   body** regardless of header fields.
2. If `Transfer-Encoding` is present and the final coding is `chunked`, length
   is determined by chunked decoding.
3. If `Transfer-Encoding` is present but final coding is **not** `chunked`:
   - In a **response**, read until connection close.
   - In a **request**, this is unprocessable → **`400`** (cannot reliably
     frame; MUST close the connection).
4. If **both** `Transfer-Encoding` and `Content-Length` are present →
   `Content-Length` MUST be ignored; the message is suspect. A proxy SHOULD
   reject/close. (Primary smuggling vector — see §11.2.)
5. If `Content-Length` has multiple values or an invalid value → **`400`**.
6. Valid single `Content-Length` → that many octets.
7. Request with none of the above → length is **0**.
8. Response with none of the above → read until connection close.

**MUST NOT** send both `Transfer-Encoding` and `Content-Length`.

## Chunked Transfer Coding (§7.1)

```
chunked-body = *chunk last-chunk trailer-section CRLF
chunk        = chunk-size [ chunk-ext ] CRLF chunk-data CRLF
last-chunk   = 1*"0" [ chunk-ext ] CRLF
```

- `chunk-size` is **hex**. Reject non-hex, leading `+`/`-`/`0x`, or oversized
  sizes (integer overflow is a classic parser bug).
- `chunk-ext` is rarely used; recipients MAY ignore but MUST parse safely.
- Trailers: only fields not needed before the body (never framing/routing
  fields like `Content-Length`, `Transfer-Encoding`, `Host`).
- `Transfer-Encoding` is hop-by-hop and **HTTP/1.1 only** — MUST NOT be
  forwarded to an HTTP/1.0 recipient, and does not exist in h2/h3.

## Connection Management (§9)

- Persistent by default in HTTP/1.1. `Connection: close` signals the last
  message on the connection.
- `Connection` names hop-by-hop fields that MUST be removed before forwarding.
- Pipelining allowed for idempotent requests but fragile; nginx does not
  pipeline upstream.
- `Upgrade` + `101 Switching Protocols`: protocol transition. See
  [rfc-9931.md](./rfc-9931.md) for the optimistic-transition security rules
  that 9931 adds to this section.

## §11.2 Request Smuggling (security-critical)

Smuggling arises when two parties in a chain disagree on message boundaries:
- **CL.TE / TE.CL**: one party honors `Content-Length`, the other
  `Transfer-Encoding`.
- **TE.TE**: obfuscated `Transfer-Encoding` (e.g. `Transfer-Encoding: xchunked`,
  duplicated headers) parsed differently.
- **Bad chunk sizes**, embedded CR/LF, or whitespace tricks in the request
  line/headers.

Defenses nginx must uphold:
- Reject messages with both CL and TE (or normalize to reject).
- Reject invalid/duplicate `Content-Length`.
- Reject non-`chunked` final `Transfer-Encoding` on requests with `400` +
  close.
- Do not accept bare CR or obs-fold; require proper `CRLF`.
- Ensure proxy parsing matches what it forwards.

## nginx Implementation Pointers

| Area | nginx source |
|------|-------------|
| Request line parser | `src/http/ngx_http_parse.c` `ngx_http_parse_request_line()` |
| Header parser | `ngx_http_parse_header_line()` |
| Chunked decode | `ngx_http_parse_chunked()` |
| Body length / discard | `src/http/ngx_http_request_body.c` |
| `Content-Length`/`Transfer-Encoding` validation | `src/http/ngx_http_request.c` (`ngx_http_process_request_header`) |
| Connection/keepalive | `src/http/ngx_http_request.c` (`ngx_http_set_keepalive`) |
| Status line output | `src/http/ngx_http_header_filter_module.c` |

## Compatibility

9112 replaces the message-syntax/routing/framing portions of **7230** (which
obsoleted **2616 §3–§6**). The 9110/9112 split separated *semantics* from
*wire format*, which 2616 combined. See [compatibility.md](./compatibility.md).
