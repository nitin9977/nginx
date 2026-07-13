# HTTP RFC Compatibility & Obsoletion Map

Reference for reading legacy nginx code, old issue tickets, CVE advisories, and
third-party specs that still cite retired HTTP RFC numbers. The current core
(9110–9112, June 2022) reorganized the material, so section numbers do **not**
line up one-to-one with the old RFCs.

## Lineage Timeline

```
RFC 1945 (1996)  HTTP/1.0  — informational, first written spec
      │
RFC 2068 (1997)  HTTP/1.1  — first HTTP/1.1  (obsoleted by 2616)
      │
RFC 2616 (1999)  HTTP/1.1  — the long-lived monolith (obsoleted by 7230-7235)
      │
RFC 7230-7235 (2014)  "723x" — split into 6 documents (obsoleted by 9110-9112)
      │
RFC 9110/9111/9112 (2022)  — current core (semantics / caching / HTTP/1.1)
```

HTTP/2: RFC 7540 (2015) → **RFC 9113** (2022).
HTTP/3: **RFC 9114** (2022), new — no predecessor.

## The 723x Series → Current Core

| Old RFC | Title (2014) | Obsoleted by | Where the content lives now |
|---------|--------------|--------------|-----------------------------|
| **7230** | Message Syntax and Routing | 9110 + 9112 | Framing/wire format → **9112**; routing, `Host`, `Via`, field ABNF → **9110** §5–§7 |
| **7231** | Semantics and Content | 9110 | Methods, status codes, representation, negotiation → **9110** §8–§12, §15 |
| **7232** | Conditional Requests | 9110 | `ETag`, `If-*`, precedence → **9110** §8.8, §13 |
| **7233** | Range Requests | 9110 | `Range`, `206`, `416` → **9110** §14 |
| **7234** | Caching | 9111 | Entire caching model → **9111** |
| **7235** | Authentication | 9110 | `WWW-Authenticate`, `Authorization`, `401`/`407` → **9110** §11 |

RFC 7236 (auth scheme registrations) and 7237 (method registrations) fed the
IANA registries now maintained per RFC 9110 §16.

## RFC 2616 (single-document HTTP/1.1) → Current

RFC 2616 combined semantics + framing + caching + auth in one document. Rough
mapping of its well-known sections:

| RFC 2616 § | Topic | Now in |
|-----------|-------|--------|
| §3 | Protocol parameters (versions, date, charset, codings) | 9110 §5–§8, 9112 §2 |
| §4 | HTTP message | 9112 §2–§6 |
| §5 | Request | 9112 §3, 9110 §7 |
| §6 | Response | 9112 §4, 9110 §15 |
| §7 | Entity | 9110 §8 (representations) |
| §8 | Connections | 9112 §9 |
| §9 | Method definitions | 9110 §9 |
| §10 | Status code definitions | 9110 §15 |
| §13 | Caching | 9111 |
| §14 | Header field definitions | 9110 §5–§14 (by topic), 9112 (framing fields) |
| §15 | Security considerations | 9110 §17, 9112 §11 |

## RFC 1945 (HTTP/1.0) — still relevant for interop

- No mandatory `Host` header; single request per connection by default
  (persistence was an extension).
- No chunked transfer coding — body length is by `Content-Length` or
  connection close only.
- `Transfer-Encoding` MUST NOT be sent to an HTTP/1.0 recipient (9112 §7).
- Limited status codes; treat unknown ones by class.
- nginx downgrades to HTTP/1.0 framing rules when the client/upstream reports
  `HTTP/1.0` — relevant when debugging keepalive or chunked issues.

## Terminology Changes Worth Knowing

| Old term (2616/723x) | Current term (9110) |
|----------------------|---------------------|
| entity / entity-header | representation / representation metadata |
| entity-body | content |
| message-header | field (field-line, header field) |
| request-URI | request-target / target URI |
| Warning header | deprecated (removed from 9111) |
| `Content-MD5` | removed |
| persistent connection | connection (persistence is the HTTP/1.1 default) |

## Practical Guidance

- When a **CVE or ticket cites RFC 7230 §3.3.3**, that is now **RFC 9112 §6**
  (message body length) — the request-smuggling framing algorithm.
- When old nginx comments reference "RFC 2616 §14.x", translate via the table
  above before assuming behavior; several requirements were tightened in the
  2022 rewrite (e.g. obs-fold, dual `Content-Length`/`Transfer-Encoding`).
- Requirement levels can differ: some 2616 `SHOULD`s became `MUST`s in 9110–
  9112. Always confirm against the current RFC text before citing normative
  strength.
