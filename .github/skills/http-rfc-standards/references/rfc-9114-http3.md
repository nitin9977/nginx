# RFC 9114 — HTTP/3

> Bishop, Ed. — June 2022.
> Full text: <https://www.rfc-editor.org/rfc/rfc9114.txt>

HTTP semantics (RFC 9110) mapped onto **QUIC** (RFC 9000) transport. QUIC
provides stream multiplexing, per-stream reliability, and integrated TLS 1.3,
which removes HTTP/2's head-of-line blocking at the transport layer. Maps to
`src/http/v3/` and `src/event/quic/` in nginx.

## Section Map

| § | Topic |
|---|-------|
| 3 | Connection setup — ALPN `h3`, QUIC transport parameters, discovery via `Alt-Svc` |
| 4 | Expressing HTTP semantics — request/response on bidirectional streams, field formatting, pseudo-headers, `CONNECT` |
| 5 | Connection closure — GOAWAY, connection errors, draining |
| 6 | Stream mapping & usage — bidirectional (requests) vs unidirectional (control, QPACK, push) streams |
| 7 | HTTP framing layer — frame format and DATA/HEADERS/etc. |
| 8 | Error handling & error codes |
| 9 | Extensions to HTTP/3 |
| 10 | Security considerations |
| 11 | IANA (frame types, settings, error codes, stream types) |

## QUIC Stream Model (§6)

- **Client request** = one client-initiated **bidirectional** stream carrying
  a HEADERS frame (+ optional DATA + optional trailing HEADERS).
- **Unidirectional streams** carry a type prefix:
  - Control stream (SETTINGS, GOAWAY, etc.) — exactly one per peer.
  - QPACK encoder stream & QPACK decoder stream.
  - Push stream (server push).
- Because QUIC handles per-stream ordering/loss, HTTP/3 has **no
  connection-level flow control frames** of its own — flow control is QUIC's
  job (transport parameters + MAX_DATA/MAX_STREAM_DATA).

## HTTP/3 Frames (§7)

| Frame | Purpose |
|-------|---------|
| DATA | Message content |
| HEADERS | QPACK-encoded field section |
| CANCEL_PUSH | Cancel a promised push |
| SETTINGS | Connection configuration (on the control stream only) |
| PUSH_PROMISE | Server push promise |
| GOAWAY | Graceful shutdown |
| MAX_PUSH_ID | Bound on push IDs |

- Frames use a variable-length-integer `Type` + `Length` encoding (QUIC
  varint), **not** the fixed 9-octet header of HTTP/2.
- SETTINGS parameters are HTTP/3-specific (e.g.
  `SETTINGS_QPACK_MAX_TABLE_CAPACITY`, `SETTINGS_MAX_FIELD_SECTION_SIZE`,
  `SETTINGS_QPACK_BLOCKED_STREAMS`).

## QPACK (RFC 9204)

- Header compression designed for QUIC's out-of-order delivery. Uses static +
  dynamic tables like HPACK, but with separate encoder/decoder streams to
  avoid head-of-line blocking on the dynamic table.
- `SETTINGS_QPACK_BLOCKED_STREAMS` bounds how many streams may block awaiting
  dynamic-table state.

## Message Rules (§4) — mostly identical to HTTP/2

- Same pseudo-headers: `:method`, `:scheme`, `:authority`, `:path`,
  `:status`.
- Field names MUST be lowercase; connection-specific fields (`Connection`,
  `Keep-Alive`, `Transfer-Encoding`, `Upgrade`, `Proxy-Connection`) are
  forbidden; `TE` only `trailers`.
- **No `Transfer-Encoding`**, no chunked; framing is via QUIC streams + HTTP/3
  frames.
- `CONNECT` maps to a bidirectional stream tunnel; extended CONNECT
  (RFC 9220) enables WebSockets over h3.
- HTTP/3 does **not** use `Upgrade`/`101` (relevant to rfc-9931.md, which is
  HTTP/1.1-specific).

## Security Considerations (§10)

- Inherits QUIC's TLS 1.3 security; amplification/DoS concerns handled at the
  QUIC layer (address validation, anti-amplification limit).
- HTTP/3-specific: QPACK decompression resource limits, SETTINGS/GOAWAY abuse,
  push flooding, stream-limit exhaustion.
- 0-RTT data is replayable → MUST NOT carry non-idempotent effects
  unguarded.

## nginx Implementation Pointers

| Area | nginx source |
|------|-------------|
| HTTP/3 request handling | `src/http/v3/ngx_http_v3_request.c` |
| HTTP/3 framing | `src/http/v3/ngx_http_v3_streams.c`, `ngx_http_v3_parse.c` |
| QPACK | `src/http/v3/ngx_http_v3_table.c` |
| Module / directives | `src/http/v3/ngx_http_v3_module.c` |
| QUIC transport | `src/event/quic/ngx_event_quic*.c` |

## Compatibility

HTTP/3 has no obsoleted-RFC lineage in the 723x/2616 series — it shares only
*semantics* (RFC 9110) with earlier HTTP versions. It depends on QUIC
(RFC 9000/9001/9002) and QPACK (RFC 9204). `Alt-Svc` (RFC 7838) is the usual
discovery path from an HTTP/1.1 or HTTP/2 origin.
