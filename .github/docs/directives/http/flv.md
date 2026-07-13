# Module ngx_http_flv_module

**Source**: https://nginx.org/en/docs/http/ngx_http_flv_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives flv The ngx_http_flv_module module provides pseudo-streaming server-side support for Flash Video (FLV) files. It handles requests with the start argument in the request URI’s query string specially, by sending back the contents of a file starting from the requested byte offset and with the prepended FLV header. This module is not built by default, it should be enabled with the --with-http_flv_module configuration parameter.

## Example Configuration

```nginx
location ~ \.flv$ {
    flv;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `flv` | flv ; | — | location |

## Directive Details

### `flv`

**Syntax**: `flv ;`  
**Default**: `—`  
**Context**: `location`  

Turns on module processing in a surrounding location.
