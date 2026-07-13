---
name: http-rfc-standards
description: 'HTTP protocol standards knowledge base (IETF RFCs). USE FOR: implementing or reviewing nginx HTTP behavior against the specs — request/response semantics, methods, status codes, header field syntax, message parsing, chunked transfer, conditional requests, ranges, caching (Cache-Control/Vary/freshness), connection management, HTTP/1.1 message framing, HTTP/2 (h2) framing/HPACK/streams, HTTP/3 (h3/QUIC) framing/QPACK, Upgrade/CONNECT protocol transitions, request smuggling defenses. Covers RFC 9110 (Semantics), 9111 (Caching), 9112 (HTTP/1.1), 9113 (HTTP/2), 9114 (HTTP/3), 9931 (optimistic protocol transition security). For backward compatibility also maps to obsoleted RFC 7230-7235 (723x), 2616, 2068, and 1945 (HTTP/1.0). DO NOT USE FOR: nginx config directive syntax (use nginx-internals directive reference), non-HTTP protocols. INVOKES: read_file for reference lookups, webfetch for authoritative RFC text at rfc-editor.org.'
argument-hint: 'Describe the HTTP behavior, header, status code, or framing question to check against the RFCs'
---

# HTTP RFC Standards Knowledge Base

Distilled, implementation-focused summaries of the current IETF HTTP core
specifications and their obsoleted predecessors. Use this skill to check
nginx behavior against the normative requirements without re-reading the
full RFC text on every prompt.

> **Authoritative text**: Every RFC in this skill is published at
> `https://www.rfc-editor.org/rfc/rfcNNNN.txt` (or `.html`). When a
> requirement's exact wording matters (MUST/MUST NOT/SHOULD), fetch the
> section from rfc-editor.org rather than relying on the summary.

## The Current HTTP Core Suite (June 2022+)

The "HTTP core" was re-published in June 2022 as an internally consistent
set. All prior HTTP core RFCs (2616, 7230–7235) are **obsoleted** by these:

| RFC | STD | Title | Scope |
|-----|-----|-------|-------|
| [9110](./references/rfc-9110-semantics.md) | STD 97 | HTTP Semantics | Version-independent: methods, status codes, header fields, content, conditionals, ranges, auth |
| [9111](./references/rfc-9111-caching.md) | STD 98 | HTTP Caching | Freshness, validation, `Cache-Control`, `Vary`, invalidation |
| [9112](./references/rfc-9112-http11.md) | STD 99 | HTTP/1.1 | Wire format: message framing, chunked transfer, connection management |
| [9113](./references/rfc-9113-http2.md) | — | HTTP/2 | Binary framing, streams, multiplexing, HPACK, flow control |
| [9114](./references/rfc-9114-http3.md) | — | HTTP/3 | HTTP over QUIC, QPACK, unidirectional streams |
| [9931](./references/rfc-9931.md) | — | Optimistic Protocol Transitions (security) | Updates 9112 + 9298: Upgrade/CONNECT request-smuggling defenses |

**Compatibility / obsoleted specs**:
[compatibility.md](./references/compatibility.md) maps RFC 9110–9112 back to
RFC **7230–7235** (the "723x" series), RFC **2616**, RFC **2068**, and RFC
**1945** (HTTP/1.0). Use it when reading old code, old tickets, or specs that
still cite the retired RFC numbers.

## When to Use

- Implementing or reviewing HTTP request/response handling in nginx C source
- Deciding whether a status code, method, or header behavior is spec-compliant
- Auditing message framing: `Content-Length` vs `Transfer-Encoding: chunked`
- Reviewing cache semantics (`Cache-Control`, `Age`, `Expires`, `Vary`, `ETag`)
- Working on `ngx_http_v2_*` (HTTP/2) or `ngx_http_v3_*` / QUIC code
- Analyzing request-smuggling / desync vulnerabilities (see 9112 §11.2, 9931)
- Reviewing `Upgrade`, `101 Switching Protocols`, or `CONNECT` handling
- Reading legacy code/CVEs that reference RFC 2616 or 7230–7235 section numbers

## Procedure

1. **Identify the layer**: semantics (9110), wire format (9112/9113/9114), or
   caching (9111). Most header-field *meaning* questions are 9110; most
   *framing* questions are version-specific.
2. **Open the matching reference** from the table above for the section map,
   key requirements, and nginx source pointers.
3. **Confirm exact normative wording** for any MUST/SHOULD you will cite:
   `webfetch https://www.rfc-editor.org/rfc/rfcNNNN.txt` and read the section.
4. **Map to nginx source** using the "nginx implementation" pointers in each
   reference, then verify with `read_file` (see the
   [nginx-internals](../nginx-internals/SKILL.md) module map).
5. **Check compatibility** if the code or ticket cites an old RFC number —
   translate via [compatibility.md](./references/compatibility.md).

## Key Cross-Cutting Rules (Always True)

- **ABNF is authoritative**: field syntax uses the ABNF in RFC 9110 §5 and
  the collected grammars. `token`, `OWS`, `field-name`, `field-value` are
  defined once in 9110 and reused everywhere.
- **Semantics are version-independent**: a `GET`, a `304`, or an `ETag` means
  the same thing over HTTP/1.1, /2, and /3. Only the *framing* differs.
- **Framing determines message length** and is the #1 security surface:
  disagreements between `Content-Length` and `Transfer-Encoding`, or between
  two parsers, cause request smuggling (9112 §11.2, 9931).
- **`Transfer-Encoding` is HTTP/1.1-only**: it MUST NOT appear in HTTP/2 or
  HTTP/3; those use frame-based framing instead (9113 §8.2.2, 9114 §4.1).
- **Connection-specific header fields** (`Connection`, `Keep-Alive`,
  `Transfer-Encoding`, `Upgrade`, `Proxy-Connection`) MUST NOT be forwarded
  and MUST NOT appear in h2/h3 (9113 §8.2.2).
- **Caches key on method + URI + `Vary`**; a response is reusable only if
  fresh or successfully validated (9111 §4).
- **Never trust a single parser**: when nginx is a proxy, its parsing must
  agree with the upstream's; ambiguity must be rejected, not guessed.

## Quick Lookup: "Which RFC Owns X?"

| Topic | RFC | Reference |
|-------|-----|-----------|
| Methods (GET/POST/CONNECT/…) | 9110 §9 | rfc-9110-semantics.md |
| Status codes (1xx–5xx) | 9110 §15 | rfc-9110-semantics.md |
| Header field definitions & ABNF | 9110 §5, §6 | rfc-9110-semantics.md |
| Representation data, `Content-*` | 9110 §8 | rfc-9110-semantics.md |
| Conditional requests (`If-*`, `ETag`) | 9110 §13 | rfc-9110-semantics.md |
| Range requests (`Range`, `206`, `416`) | 9110 §14 | rfc-9110-semantics.md |
| Authentication (`WWW-Authenticate`) | 9110 §11 | rfc-9110-semantics.md |
| Caching, freshness, `Cache-Control` | 9111 | rfc-9111-caching.md |
| HTTP/1.1 message parsing / start-line | 9112 §2–§3 | rfc-9112-http11.md |
| Chunked transfer coding | 9112 §7 | rfc-9112-http11.md |
| Connection mgmt / persistence | 9112 §9 | rfc-9112-http11.md |
| Request smuggling | 9112 §11.2 | rfc-9112-http11.md |
| HTTP/2 frames / HPACK / flow control | 9113 | rfc-9113-http2.md |
| HTTP/3 frames / QPACK / QUIC streams | 9114 | rfc-9114-http3.md |
| Upgrade / CONNECT / 101 security | 9931 | rfc-9931.md |
| Old RFC section-number mapping | — | compatibility.md |
