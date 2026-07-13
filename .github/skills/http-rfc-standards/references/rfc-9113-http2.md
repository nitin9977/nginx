# RFC 9113 — HTTP/2

> Thomson, Benfield, Eds. — June 2022. Obsoletes 7540 and 8740.
> Full text: <https://www.rfc-editor.org/rfc/rfc9113.txt>

Binary, multiplexed framing for HTTP semantics (RFC 9110) over a single TCP
(usually TLS) connection. Maps to `src/http/v2/` in nginx
(`ngx_http_v2.c`, `ngx_http_v2_filter_module.c`, `ngx_http_v2_table.c`).

## Section Map

| § | Topic |
|---|-------|
| 3 | Connection establishment — ALPN `h2`, HTTP/1.1 `Upgrade` (deprecated), prior knowledge, the connection preface |
| 4 | Frame format, header compression basics, frame size |
| 5 | Streams & multiplexing — stream states, identifiers, concurrency, priority (deprecated), flow-control window, error handling |
| 6 | Frame definitions — DATA, HEADERS, PRIORITY, RST_STREAM, SETTINGS, PUSH_PROMISE, PING, GOAWAY, WINDOW_UPDATE, CONTINUATION |
| 7 | Error codes |
| 8 | HTTP message exchange — mapping requests/responses onto streams, pseudo-headers, connection-specific fields, `CONNECT`, `Upgrade`/`101` not used |
| 9 | Additional requirements — TLS usage, `h2c` |
| 10 | Security considerations |

## Framing (§4, §6)

- Fixed 9-octet frame header: `Length(24) Type(8) Flags(8) R(1)
  StreamID(31)`.
- Default max frame payload = 16,384 (2^14); peers negotiate larger via
  `SETTINGS_MAX_FRAME_SIZE`.
- Frame types:

| Type | Purpose |
|------|---------|
| DATA | Message content |
| HEADERS | Opens a stream, carries an HPACK header block |
| PRIORITY | Deprecated in 9113 (was 7540 tree) |
| RST_STREAM | Abrupt stream termination + error code |
| SETTINGS | Connection configuration (must be ACKed) |
| PUSH_PROMISE | Server push (disabled by `SETTINGS_ENABLE_PUSH = 0`) |
| PING | RTT / keepalive |
| GOAWAY | Graceful connection shutdown + last processed stream |
| WINDOW_UPDATE | Flow-control credit |
| CONTINUATION | Header-block fragment continuation |

## Streams (§5)

- Client streams are **odd**, server-initiated (push) are **even**; IDs are
  monotonically increasing and never reused.
- Stream states: idle → open → half-closed → closed (plus reserved for push).
- `SETTINGS_MAX_CONCURRENT_STREAMS` bounds parallelism.
- **Flow control** is credit-based, per-stream and per-connection, via
  `WINDOW_UPDATE`; only DATA frames are flow-controlled. Initial window =
  65,535 octets.

## HPACK (RFC 7541)

- Header compression using a static table + dynamic table + Huffman coding.
- `SETTINGS_HEADER_TABLE_SIZE` bounds the dynamic table.
- Security: guard against decompression bombs and dynamic-table poisoning.

## Message Mapping (§8) — key compliance rules

- **Pseudo-header fields** (must precede regular fields, request only unless
  noted): `:method`, `:scheme`, `:authority`, `:path` (request);
  `:status` (response). Unknown/misplaced pseudo-headers → malformed → treat
  as stream error (`PROTOCOL_ERROR`).
- **No `Transfer-Encoding`**; framing is via frames. A `Content-Length`, if
  present, must match the DATA length.
- **Connection-specific header fields are forbidden**: `Connection`,
  `Proxy-Connection`, `Keep-Alive`, `Transfer-Encoding`, `Upgrade` MUST NOT
  appear. `TE` MAY only carry the value `trailers`.
- Field names MUST be lowercase; uppercase → malformed.
- `Host` is optional if `:authority` is present; if both, they must agree.
- `CONNECT` uses `:method = CONNECT` with only `:authority`; establishes a
  tunnel on that stream.
- HTTP/2 does **not** use `Upgrade`/`101`; the h2→other transition mechanism
  of HTTP/1.1 does not exist here (relevant to rfc-9931.md scope).

## Security Considerations (§10)

- Resource-exhaustion via many small frames, CONTINUATION floods, empty DATA
  frames, RST_STREAM floods (cf. "HTTP/2 Rapid Reset", CVE-2023-44487),
  SETTINGS floods, window manipulation.
- CVE-2023-44487 (Rapid Reset): unbounded stream open+RST cycles; mitigate by
  capping resets per connection. nginx: see `ngx_http_v2` reset accounting.
- 9113 tightened several 7540 ambiguities (field validation, `:authority`).

## nginx Implementation Pointers

| Area | nginx source |
|------|-------------|
| Connection / frame state machine | `src/http/v2/ngx_http_v2.c` |
| HPACK dynamic table | `src/http/v2/ngx_http_v2_table.c` |
| Response framing / output | `src/http/v2/ngx_http_v2_filter_module.c` |
| Module config / directives | `src/http/v2/ngx_http_v2_module.c` |
| Pseudo-header + field validation | `ngx_http_v2_*` in `ngx_http_v2.c` |

## Compatibility

9113 obsoletes **7540** (original HTTP/2) and **8740** (an errata/clarification
doc). It removed stream prioritization (deferred to RFC 9218) and tightened
header-field validation. HPACK remains defined by **7541** (unchanged). No
HTTP/1.x compatibility mapping applies — h2 shares only *semantics* (9110)
with older versions.
