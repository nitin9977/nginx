# Module ngx_http_stub_status_module

**Source**: https://nginx.org/en/docs/http/ngx_http_stub_status_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives stub_status Data Embedded Variables The ngx_http_stub_status_module module provides access to basic status information. This module is not built by default, it should be enabled with the --with-http_stub_status_module configuration parameter.

## Example Configuration

```nginx
location = /basic_status {
    stub_status;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `stub_status` | stub_status ; | — | server , location |

## Directive Details

### `stub_status`

**Syntax**: `stub_status ;`  
**Default**: `—`  
**Context**: `server , location`  

The basic status information will be accessible from the surrounding location.
