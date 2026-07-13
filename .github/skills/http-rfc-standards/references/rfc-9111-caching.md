# RFC 9111 — HTTP Caching (STD 98)

> Fielding, Nottingham, Reschke, Eds. — June 2022. Obsoletes 7234.
> Full text: <https://www.rfc-editor.org/rfc/rfc9111.txt>

Defines when and how HTTP responses may be **stored and reused**. Relevant to
nginx `proxy_cache`, `fastcgi_cache`, `scgi_cache`, `uwsgi_cache`, and the
`Expires`/`Cache-Control` handling in the header filter.

## Section Map

| § | Topic |
|---|-------|
| 3 | Storing responses in caches — what MAY be stored |
| 4 | Constructing responses from caches — freshness, validation, `Vary`, selection |
| 5 | Field definitions — `Age`, `Cache-Control`, `Expires`, `Pragma`, `Warning` (deprecated) |
| 6 | History lists |
| 7 | Security considerations |

## What a Cache MAY Store (§3)

A cache MUST NOT store a response unless **all** hold:
- The request method is understood and defined as cacheable (GET/HEAD, or POST
  with explicit freshness).
- The response status code is understood.
- `no-store` is absent (in request and response `Cache-Control`).
- `private` is absent, OR the cache is private (not shared).
- For shared caches: the request has no `Authorization`, OR the response
  explicitly allows it (`must-revalidate`, `public`, or `s-maxage`).
- The response contains at least one of: explicit expiry (`Expires`,
  `max-age`, `s-maxage`), a cache-control extension permitting it, a
  cacheable status code, or `public`.

## Freshness (§4.2)

- **Fresh** if `age < freshness_lifetime`. Fresh responses are reused without
  contacting the origin.
- Freshness lifetime precedence:
  1. `s-maxage` (shared caches only)
  2. `max-age`
  3. `Expires` − `Date`
  4. Heuristic (only if none of the above and status is heuristically
     cacheable; e.g. 10% of (Date − Last-Modified)).
- **Age calculation** (§4.2.3): `Age` accumulates resident time + transit
  time; `apparent_age` and `corrected_age_value` guard against clock skew.

## Validation (§4.3)

- A stale response may be reused after **validation**: send a conditional
  request (`If-None-Match` with stored `ETag`, `If-Modified-Since` with stored
  `Last-Modified`).
- `304 Not Modified` → update stored headers, reuse stored content.
- Full `200` → replace the stored response.
- `no-cache` means "may store, but MUST revalidate before reuse".

## `Vary` and Response Selection (§4.1)

- A stored response with `Vary` matches a new request only if the nominated
  request header fields match (normalized). `Vary: *` = never matches (always
  revalidate/refetch).
- The cache key is effectively **method + target URI + selecting header
  fields listed in `Vary`**.

## `Cache-Control` Directives (§5.2)

**Request directives**: `max-age`, `max-stale`, `min-fresh`, `no-cache`,
`no-store`, `no-transform`, `only-if-cached`.

**Response directives**: `max-age`, `s-maxage`, `no-cache`, `no-store`,
`no-transform`, `must-revalidate`, `proxy-revalidate`, `must-understand`,
`private`, `public`, `immutable` (informational), `stale-while-revalidate`
and `stale-if-error` (RFC 5861 extensions).

- `must-revalidate` / `proxy-revalidate`: once stale, MUST NOT serve without
  successful validation (return `504` if origin unreachable).
- `no-store` on request or response forbids storage of that message.

## Invalidation (§4.4)

- A non-error response to an **unsafe** method (POST/PUT/DELETE/PATCH)
  invalidates the target URI, and the URIs in `Location` / `Content-Location`
  if same-origin.

## nginx Implementation Pointers

| Area | nginx source |
|------|-------------|
| Cache core / key / freshness | `src/http/ngx_http_cache.h`, `src/http/ngx_http_file_cache.c` |
| Upstream cache integration | `src/http/ngx_http_upstream.c` (`ngx_http_upstream_process_cache_control`, `_expires`, `_vary`) |
| `Expires`/`Cache-Control` output | `src/http/modules/ngx_http_headers_filter_module.c` |
| Conditional revalidation | `src/http/modules/ngx_http_not_modified_filter_module.c` |

> nginx does not implement RFC 9111 caching literally — directives like
> `proxy_cache_valid`, `proxy_cache_use_stale`, and `proxy_ignore_headers`
> intentionally override or bypass parts of the spec. When reviewing cache
> code, check whether observed behavior is nginx policy vs. an RFC violation.

## Compatibility

9111 obsoletes **7234** (which obsoleted the caching part of **2616 §13**).
`Warning` header handling from 7234 is now deprecated. See
[compatibility.md](./compatibility.md).
