# Module ngx_stream_js_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_js_module.html  

## Overview

Example Configuration Directives js_access js_context_reuse js_engine js_fetch_buffer_size js_fetch_ciphers js_fetch_max_response_buffer_size js_fetch_protocols js_fetch_timeout js_fetch_trusted_certificate js_fetch_verify js_fetch_verify_depth js_fetch_proxy js_fetch_keepalive js_fetch_keepalive_requests js_fetch_keepalive_time js_fetch_keepalive_timeout js_filter js_import js_include js_load_stream_native_module js_path js_periodic js_preload_object js_preread js_set js_shared_dict_zone js_var Session Object Properties The ngx_stream_js_module module is used to implement handlers in njs — a subset of the JavaScript language. Download and install instructions are available here .

## Example Configuration

```nginx
stream {
    # since 0.9.1
    js_engine qjs;

    js_import stream.js;

    js_set $bar stream.bar;
    js_set $req_line stream.req_line;

    server {
        listen 12345;

        js_preread stream.preread;
        return     $req_line;
    }

    server {
        listen 12346;

        js_access  stream.access;
        proxy_pass 127.0.0.1:8000;
        js_filter  stream.header_inject;
    }
}

http {
    server {
        listen 8000;
        location / {
            return 200 $http_foo\n;
        }
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `js_access` | js_access module.function ; | — | stream , server |
| `js_filter` | js_filter module.function ; | — | stream , server |
| `js_include` | js_include file ; | — | stream |
| `js_preread` | js_preread module.function ; | — | stream , server |
| `js_set` | js_set $variable module.function [ nocache ]; | — | stream , server |

## Directive Details

### `js_access`

**Syntax**: `js_access module.function ;`  
**Default**: `—`  
**Context**: `stream , server`  

Sets an njs function which will be called at the access phase. Since 0.4.0 , a module function can be referenced. The function is called once at the moment when the stream session reaches the access phase for the first time. The function is called with the following arguments:

### `js_filter`

**Syntax**: `js_filter module.function ;`  
**Default**: `—`  
**Context**: `stream , server`  

Sets a data filter. Since 0.4.0 , a module function can be referenced. The filter function is called once at the moment when the stream session reaches the content phase. The filter function is called with the following arguments:

### `js_include`

**Syntax**: `js_include file ;`  
**Default**: `—`  
**Context**: `stream`  

Specifies a file that implements server and variable handlers in njs: nginx.conf: js_include stream.js; js_set $js_addr address; server { listen 127.0.0.1:12345; return $js_addr; } stream.js: function address(s) { return s.remoteAddress; } The directive was made obsolete in version 0.4.0 and was removed in version 0.7.1 . The js_import directive should be used instead.

### `js_preread`

**Syntax**: `js_preread module.function ;`  
**Default**: `—`  
**Context**: `stream , server`  

Sets an njs function which will be called at the preread phase. Since 0.4.0 , a module function can be referenced. The function is called once at the moment when the stream session reaches the preread phase for the first time. The function is called with the following arguments:

### `js_set`

**Syntax**: `js_set $variable module.function [ nocache ];`  
**Default**: `—`  
**Context**: `stream , server`  

Sets an njs function for the specified variable . Since 0.4.0 , a module function can be referenced. The function is called when the variable is referenced for the first time for a given request. The exact moment depends on a phase at which the variable is referenced. This can be used to perform some logic not related to variable evaluation. For example, if the variable is referenced only in the log_format directive, its handler will not be executed until the log phase. This handler can be used to do some cleanup right before the request is freed. Since 0.8.6 , when optional argument nocache is provided the handler is called every time it is referenced. Due to current limitations of the rewrite module, when a nocache variable is referenced by the set directive its handler should always return a fixed-length value.
