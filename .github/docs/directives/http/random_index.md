# Module ngx_http_random_index_module

**Source**: https://nginx.org/en/docs/http/ngx_http_random_index_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives random_index The ngx_http_random_index_module module processes requests ending with the slash character (‘ / ’) and picks a random file in a directory to serve as an index file. The module is processed before the ngx_http_index_module module. This module is not built by default, it should be enabled with the --with-http_random_index_module configuration parameter.

## Example Configuration

```nginx
location / {
    random_index on;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `random_index` | random_index on \| off ; | random_index off; | location |

## Directive Details

### `random_index`

**Syntax**: `random_index on | off ;`  
**Default**: `random_index off;`  
**Context**: `location`  

Enables or disables module processing in a surrounding location.
