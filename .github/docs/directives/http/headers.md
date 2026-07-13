# Module ngx_http_headers_module

**Source**: https://nginx.org/en/docs/http/ngx_http_headers_module.html  

## Overview

Example Configuration Directives add_header add_header_inherit add_trailer add_trailer_inherit expires The ngx_http_headers_module module allows adding the “Expires” and “Cache-Control” header fields, and arbitrary fields, to a response header.

## Example Configuration

```nginx
expires    24h;
expires    modified +24h;
expires    @24h;
expires    0;
expires    -1;
expires    epoch;
expires    $expires;
add_header Cache-Control private;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `add_header` | add_header name value [ always ]; | — | http , server , location , if in location |
| `expires` | expires [ modified ] time ; expires epoch \| max \| off ; | expires off; | http , server , location , if in location |

## Directive Details

### `add_header`

**Syntax**: `add_header name value [ always ];`  
**Default**: `—`  
**Context**: `http , server , location , if in location`  

Adds the specified field to a response header provided that the response code equals 200, 201 (1.3.10), 204, 206, 301, 302, 303, 304, 307 (1.1.16, 1.0.13), or 308 (1.13.0). Parameter value can contain variables.

### `expires`

**Syntax**: `expires [ modified ] time ; expires epoch | max | off ;`  
**Default**: `expires off;`  
**Context**: `http , server , location , if in location`  

Enables or disables adding or modifying the “Expires” and “Cache-Control” response header fields provided that the response code equals 200, 201 (1.3.10), 204, 206, 301, 302, 303, 304, 307 (1.1.16, 1.0.13), or 308 (1.13.0). The parameter can be a positive or negative time . The time in the “Expires” field is computed as a sum of the current time and time specified in the directive. If the modified parameter is used (0.7.0, 0.6.32) then the time is computed as a sum of the file’s modification time and the time specified in the directive. In addition, it is possible to specify a time of day using the “ @ ” prefix (0.7.9, 0.6.34):
