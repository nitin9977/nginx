# Module ngx_stream_upstream_hc_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_upstream_hc_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives health_check health_check_timeout match The ngx_stream_upstream_hc_module module (1.9.0) allows enabling periodic health checks of the servers in a group . The server group must reside in the shared memory . If a health check fails, the server will be considered unhealthy. If several health checks are defined for the same group of servers, a single failure of any check will make the corresponding server be considered unhealthy. Client connections are not passed to unhealthy servers and servers in the “checking” state. This module is available as part of our commercial subscription .

## Example Configuration

```nginx
upstream tcp {
    zone upstream_tcp 64k;

    server backend1.example.com:12345 weight=5;
    server backend2.example.com:12345 fail_timeout=5s slow_start=30s;
    server 192.0.2.1:12345            max_fails=3;

    server backup1.example.com:12345  backup;
    server backup2.example.com:12345  backup;
}

server {
    listen     12346;
    proxy_pass tcp;
    health_check;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `health_check` | health_check [ parameters ]; | — | server |
| `health_check_timeout` | health_check_timeout timeout ; | health_check_timeout 5s; | stream , server |
| `match` | match name { ... } | — | stream |

## Directive Details

### `health_check`

**Syntax**: `health_check [ parameters ];`  
**Default**: `—`  
**Context**: `server`  

Enables periodic health checks of the servers in a group . The following optional parameters are supported: sets the initial “checking” state for a server until the first health check is completed (1.11.7). Client connections are not passed to servers in the “checking” state. If the parameter is not specified, the server will be initially considered healthy.

### `health_check_timeout`

**Syntax**: `health_check_timeout timeout ;`  
**Default**: `health_check_timeout 5s;`  
**Context**: `stream , server`  

Overrides the proxy_timeout value for health checks.

### `match`

**Syntax**: `match name { ... }`  
**Default**: `—`  
**Context**: `stream`  

Defines the named test set used to verify server responses to health checks. The following parameters can be configured: Both send and expect parameters can contain hexadecimal literals with the prefix “ \x ” followed by two hex digits, for example, “ \x80 ” (1.9.12).
