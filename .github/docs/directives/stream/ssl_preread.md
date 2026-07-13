# Module ngx_stream_ssl_preread_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives ssl_preread Embedded Variables The ngx_stream_ssl_preread_module module (1.11.5) allows extracting information from the ClientHello message without terminating SSL/TLS, for example, the server name requested through SNI or protocols advertised in ALPN . This module is not built by default, it should be enabled with the --with-stream_ssl_preread_module configuration parameter.

## Example Configuration

```nginx
map $ssl_preread_server_name $name {
    backend.example.com      backend;
    default                  backend2;
}

upstream backend {
    server 192.168.0.1:12345;
    server 192.168.0.2:12345;
}

upstream backend2 {
    server 192.168.0.3:12345;
    server 192.168.0.4:12345;
}

server {
    listen      12346;
    proxy_pass  $name;
    ssl_preread on;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `ssl_preread` | ssl_preread on \| off ; | ssl_preread off; | stream , server |

## Directive Details

### `ssl_preread`

**Syntax**: `ssl_preread on | off ;`  
**Default**: `ssl_preread off;`  
**Context**: `stream , server`  

Enables extracting information from the ClientHello message at the preread phase.
