# Module ngx_http_upstream_hc_module

**Source**: https://nginx.org/en/docs/http/ngx_http_upstream_hc_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives health_check match The ngx_http_upstream_hc_module module allows enabling periodic health checks of the servers in a group referenced in the surrounding location. The server group must reside in the shared memory . If a health check fails, the server will be considered unhealthy. If several health checks are defined for the same group of servers, a single failure of any check will make the corresponding server be considered unhealthy. Client requests are not passed to unhealthy servers and servers in the “checking” state. Please note that most of the variables will have empty values when used with health checks. This module is available as part of our commercial subscription .

## Example Configuration

```nginx
upstream dynamic {
    zone upstream_dynamic 64k;

    server backend1.example.com      weight=5;
    server backend2.example.com:8080 fail_timeout=5s slow_start=30s;
    server 192.0.2.1                 max_fails=3;

    server backup1.example.com:8080  backup;
    server backup2.example.com:8080  backup;
}

server {
    location / {
        proxy_pass http://dynamic;
        health_check;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `health_check` | health_check [ parameters ]; | — | location |
| `match` | match name { ... } | — | http |

## Directive Details

### `health_check`

**Syntax**: `health_check [ parameters ];`  
**Default**: `—`  
**Context**: `location`  

Enables periodic health checks of the servers in a group referenced in the surrounding location. The following optional parameters are supported: sets the initial “checking” state for a server until the first health check is completed (1.11.7). Client requests are not passed to servers in the “checking” state. If the parameter is not specified, the server will be initially considered healthy.

### `match`

**Syntax**: `match name { ... }`  
**Default**: `—`  
**Context**: `http`  

Defines the named test set used to verify responses to health check requests. The following items can be tested in a response:
