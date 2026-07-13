# Module ngx_stream_set_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_set_module.html  

## Overview

Example Configuration Directives set The ngx_stream_set_module module (1.19.3) allows setting a value for a variable.

## Example Configuration

```nginx
server {
    listen 12345;
    set    $true 1;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `set` | set $variable value ; | — | server |

## Directive Details

### `set`

**Syntax**: `set $variable value ;`  
**Default**: `—`  
**Context**: `server`  

Sets a value for the specified variable . The value can contain text, variables, and their combination.
