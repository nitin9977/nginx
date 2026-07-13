# Module ngx_http_empty_gif_module

**Source**: https://nginx.org/en/docs/http/ngx_http_empty_gif_module.html  

## Overview

Example Configuration Directives empty_gif The ngx_http_empty_gif_module module emits single-pixel transparent GIF.

## Example Configuration

```nginx
location = /_.gif {
    empty_gif;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `empty_gif` | empty_gif ; | — | location |

## Directive Details

### `empty_gif`

**Syntax**: `empty_gif ;`  
**Default**: `—`  
**Context**: `location`  

Turns on module processing in a surrounding location.
