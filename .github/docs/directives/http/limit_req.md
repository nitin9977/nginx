# Module ngx_http_limit_req_module

**Source**: https://nginx.org/en/docs/http/ngx_http_limit_req_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives limit_req limit_req_dry_run limit_req_log_level limit_req_status limit_req_zone Embedded Variables The ngx_http_limit_req_module module (0.7.21) is used to limit the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address. The limitation is done using the “leaky bucket” method.

## Example Configuration

```nginx
http {
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;

    ...

    server {

        ...

        location /search/ {
            limit_req zone=one burst=5;
        }
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `limit_req` | limit_req zone = name [ burst = number ] [ nodelay \| delay = number ]; | — | http , server , location |
| `limit_req_zone` | limit_req_zone key zone = name : size rate = rate [ sync ]; | — | http |

## Directive Details

### `limit_req`

**Syntax**: `limit_req zone = name [ burst = number ] [ nodelay | delay = number ];`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets the shared memory zone and the maximum burst size of requests. If the requests rate exceeds the rate configured for a zone, their processing is delayed such that requests are processed at a defined rate. Excessive requests are delayed until their number exceeds the maximum burst size in which case the request is terminated with an error . By default, the maximum burst size is equal to zero. For example, the directives limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s; server { location /search/ { limit_req zone=one burst=5; } allow not more than 1 request per second at an average, with bursts not exceeding 5 requests. If delaying of excessive requests while requests are being limited is not desired, the parameter nodelay should be used:

### `limit_req_zone`

**Syntax**: `limit_req_zone key zone = name : size rate = rate [ sync ];`  
**Default**: `—`  
**Context**: `http`  

Sets parameters for a shared memory zone that will keep states for various keys. In particular, the state stores the current number of excessive requests. The key can contain text, variables, and their combination. Requests with an empty key value are not accounted. Usage example: limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
