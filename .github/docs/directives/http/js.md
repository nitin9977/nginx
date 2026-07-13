# Module ngx_http_js_module

**Source**: https://nginx.org/en/docs/http/ngx_http_js_module.html  

## Overview

Example Configuration Directives js_body_filter js_access js_content js_context_reuse js_engine js_fetch_buffer_size js_fetch_ciphers js_fetch_max_response_buffer_size js_fetch_protocols js_fetch_timeout js_fetch_trusted_certificate js_fetch_verify js_fetch_verify_depth js_fetch_proxy js_fetch_keepalive js_fetch_keepalive_requests js_fetch_keepalive_time js_fetch_keepalive_timeout js_header_filter js_import js_include js_load_http_native_module js_path js_periodic js_preload_object js_set js_shared_dict_zone js_var Request Argument The ngx_http_js_module module is used to implement location and variable handlers in njs — a subset of the JavaScript language. Download and install instructions are available here .

## Example Configuration

```nginx
http {
    # since 0.9.1
    js_engine qjs;

    js_import http.js;

    js_set $foo     http.foo;
    js_set $summary http.summary;
    js_set $hash    http.hash;

    resolver 10.0.0.1;

    server {
        listen 8000;

        location / {
            add_header X-Foo $foo;
            js_content http.baz;
        }

        location = /summary {
            return 200 $summary;
        }

        location = /hello {
            js_content http.hello;
        }

        # since 0.7.0
        location = /fetch {
            js_content                   http.fetch;
            js_fetch_trusted_certificate /path/to/ISRG_Root_X1.pem;
        }

        # since 0.7.0
        location = /crypto {
            add_header Hash $hash;
            return     200;
        }
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `js_content` | js_content module.function ; | — | location , if in location , limit_except |
| `js_include` | js_include file ; | — | http |
| `js_set` | js_set $variable module.function [ nocache ]; | — | http , server , location |

## Directive Details

### `js_content`

**Syntax**: `js_content module.function ;`  
**Default**: `—`  
**Context**: `location , if in location , limit_except`  

Sets an njs function as a location content handler. Since 0.4.0 , a module function can be referenced.

### `js_include`

**Syntax**: `js_include file ;`  
**Default**: `—`  
**Context**: `http`  

Specifies a file that implements location and variable handlers in njs: nginx.conf: js_include http.js; location /version { js_content version; } http.js: function version(r) { r.return(200, njs.version); } The directive was made obsolete in version 0.4.0 and was removed in version 0.7.1 . The js_import directive should be used instead.

### `js_set`

**Syntax**: `js_set $variable module.function [ nocache ];`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets an njs function for the specified variable . Since 0.4.0 , a module function can be referenced. The function is called when the variable is referenced for the first time for a given request. The exact moment depends on a phase at which the variable is referenced. This can be used to perform some logic not related to variable evaluation. For example, if the variable is referenced only in the log_format directive, its handler will not be executed until the log phase. This handler can be used to do some cleanup right before the request is freed. Since 0.8.6 , if an optional argument nocache is specified, the handler is called every time it is referenced. Due to current limitations of the rewrite module, when a nocache variable is referenced by the set directive its handler should always return a fixed-length value.
