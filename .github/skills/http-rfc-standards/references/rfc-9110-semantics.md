# RFC 9110 — HTTP Semantics (STD 97)

> Fielding, Nottingham, Reschke, Eds. — June 2022.
> Obsoletes: 2616, 7231, 7232, 7233, 7235, and parts of 7230/7234.
> Full text: <https://www.rfc-editor.org/rfc/rfc9110.txt>

The **version-independent** definition of HTTP: everything that means the same
thing over HTTP/1.1, HTTP/2, and HTTP/3. If a question is about what a method,
status code, or header field *means* (as opposed to how bytes are framed on the
wire), the answer is here.

## Section Map

| § | Topic |
|---|-------|
| 3 | Terminology & architecture (client, server, intermediary, cache, resource, representation) |
| 4 | Identifiers — URI, `http`/`https` schemes, authority |
| 5 | Field (header/trailer) syntax — names, values, `OWS`, line folding forbidden |
| 6 | Message abstraction — control data, header/trailer sections, content |
| 7 | Routing — `Host`, target URI reconstruction, intermediaries, `Via`, loops |
| 8 | Representations — `Content-Type`, `Content-Encoding`, `Content-Language`, `Content-Length`, `Content-Location`, validators |
| 9 | Methods — safe/idempotent/cacheable; GET, HEAD, POST, PUT, DELETE, CONNECT, OPTIONS, TRACE |
| 10 | Message context — request (`Expect`, `From`, `Referer`, `User-Agent`) & response (`Date`, `Server`, `Allow`, `Retry-After`) control fields |
| 11 | Authentication — `WWW-Authenticate`, `Authorization`, `Proxy-Authenticate`, challenge/credential framework |
| 12 | Content negotiation — `Accept`, `Accept-Encoding`, `Accept-Language`, proactive vs reactive |
| 13 | Conditional requests — `If-Match`, `If-None-Match`, `If-Modified-Since`, `If-Unmodified-Since`, `If-Range`, precedence |
| 14 | Range requests — `Range`, `Accept-Ranges`, `Content-Range`, `206`, `416` |
| 15 | Status codes — full 1xx–5xx registry with semantics |
| 16 | Extensibility — method/status/field/auth-scheme registries |
| 17 | Security considerations |

## Field Syntax (§5) — critical for parsers

- `field-line = field-name ":" OWS field-value OWS`
- `field-name = token`; a `token` excludes separators and control chars.
- **No obs-fold**: line folding (a `field-value` continued with leading
  whitespace) is deprecated. A recipient parsing HTTP/1.1 MUST reject or
  replace obs-fold except within `message/http` (see 9112 §5.2).
- Multiple fields with the same name = equivalent to one field with values
  joined by commas — **except `Set-Cookie`**, which cannot be combined.
- Leading/trailing `OWS` in a value is not part of the value.
- Field values are opaque octets; the field definition assigns meaning.

## Methods (§9)

| Method | Safe | Idempotent | Cacheable | Notes |
|--------|:----:|:----------:|:---------:|-------|
| GET | ✓ | ✓ | ✓ | |
| HEAD | ✓ | ✓ | ✓ | Same as GET but no content in response |
| POST | ✗ | ✗ | ✓* | Cacheable only with explicit freshness |
| PUT | ✗ | ✓ | ✗ | |
| DELETE | ✗ | ✓ | ✗ | |
| OPTIONS | ✓ | ✓ | ✗ | `*` target = server-wide |
| TRACE | ✓ | ✓ | ✗ | Must not include content |
| CONNECT | ✗ | ✗ | ✗ | Establishes a tunnel (see rfc-9931.md) |

- **Safe** = read-only intent. **Idempotent** = repeat has same effect.
- Servers MUST reject a `CONNECT` with empty/invalid port (typically `400`).

## Status Codes (§15) — quick reference

- **1xx Informational**: `100 Continue`, `101 Switching Protocols`.
- **2xx Successful**: `200`, `201`, `202`, `204 No Content`, `205`,
  `206 Partial Content`.
- **3xx Redirection**: `300`, `301`, `302`, `303 See Other`,
  `304 Not Modified`, `307 Temporary Redirect`, `308 Permanent Redirect`.
  (301/302 vs 307/308: the 30x/8 variants forbid method rewriting.)
- **4xx Client Error**: `400`, `401 Unauthorized`, `403`, `404`, `405 Method
  Not Allowed` (MUST send `Allow`), `406`, `408`, `409`, `411 Length
  Required`, `413`, `414`, `415`, `416 Range Not Satisfiable`, `421
  Misdirected Request`, `426 Upgrade Required`.
- **5xx Server Error**: `500`, `501 Not Implemented`, `502 Bad Gateway`,
  `503 Service Unavailable` (may send `Retry-After`), `504 Gateway Timeout`,
  `505 HTTP Version Not Supported`.
- Unknown status codes are treated as the `x00` of their class (`499` → `4xx`).

## Conditional Requests (§13)

- Validators: strong (`ETag`, `Last-Modified` when reliable) vs weak (`W/`
  ETag). Range requests require a strong validator (or `If-Range`).
- Precedence (§13.2.2): `If-Match` → `If-Unmodified-Since` → `If-None-Match`
  → `If-Modified-Since` → `If-Range`. `If-None-Match` overrides
  `If-Modified-Since`; `If-Match` overrides `If-Unmodified-Since`.
- Successful `If-None-Match` on GET/HEAD → `304 Not Modified`; on other
  methods → `412 Precondition Failed`.

## Range Requests (§14)

- `Range: bytes=start-end` (multiple/overlapping ranges allowed).
- Satisfiable → `206 Partial Content` with `Content-Range`; multiple ranges
  use `multipart/byteranges`.
- Unsatisfiable → `416` with `Content-Range: bytes */complete-length`.
- Server MAY ignore `Range` and return `200` with the full representation.
- `Accept-Ranges: bytes` advertises support; `none` disables.

## Authentication (§11)

- `401` + `WWW-Authenticate: <scheme> realm="..."`; client resends with
  `Authorization`. Proxy equivalents: `407` + `Proxy-Authenticate` +
  `Proxy-Authorization`.
- `Authorization` is per-origin and (per 9111) makes responses non-shared-
  cacheable unless explicitly permitted.

## nginx Implementation Pointers

| Area | nginx source |
|------|-------------|
| Request-line / header parse | `src/http/ngx_http_parse.c` |
| Header field tables & handlers | `src/http/ngx_http_request.c` |
| Status code → reason phrase | `src/http/ngx_http_header_filter_module.c` |
| Conditional (`If-*`) | `src/http/modules/ngx_http_not_modified_filter_module.c` |
| Range handling | `src/http/modules/ngx_http_range_filter_module.c` |
| Content negotiation | `src/http/modules/ngx_http_charset_filter_module.c`, upstream |
| Auth (basic) | `src/http/modules/ngx_http_auth_basic_module.c` |

## Compatibility

9110 consolidates the old **7231** (semantics/content), **7232**
(conditional), **7233** (range), **7235** (authentication), plus semantic
pieces of **7230**. See [compatibility.md](./compatibility.md) for exact
section-number mapping and RFC 2616 lineage.
