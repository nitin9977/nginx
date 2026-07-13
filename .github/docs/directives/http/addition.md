# Module ngx_http_addition_module

**Source**: https://nginx.org/en/docs/http/ngx_http_addition_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives add_before_body add_after_body addition_types The ngx_http_addition_module module is a filter that adds text before and after a response. This module is not built by default, it should be enabled with the --with-http_addition_module configuration parameter.

## Example Configuration

```nginx
location / {
    add_before_body /before_action;
    add_after_body  /after_action;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `add_before_body` | add_before_body uri ; | — | http , server , location |
| `add_after_body` | add_after_body uri ; | — | http , server , location |

## Directive Details

### `add_before_body`

**Syntax**: `add_before_body uri ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Adds the text returned as a result of processing a given subrequest before the response body. An empty string ( "" ) as a parameter cancels addition inherited from the previous configuration level.

### `add_after_body`

**Syntax**: `add_after_body uri ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Adds the text returned as a result of processing a given subrequest after the response body. An empty string ( "" ) as a parameter cancels addition inherited from the previous configuration level.
