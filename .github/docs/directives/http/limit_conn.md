# Module ngx_http_limit_conn_module

**Source**: https://nginx.org/en/docs/http/ngx_http_limit_conn_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives limit_conn limit_conn_dry_run limit_conn_log_level limit_conn_status limit_conn_zone limit_zone Embedded Variables The ngx_http_limit_conn_module module is used to limit the number of connections per the defined key, in particular, the number of connections from a single IP address. Not all connections are counted. A connection is counted only if it has a request being processed by the server and the whole request header has already been read.

## Example Configuration

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    ...

    server {

        ...

        location /download/ {
            limit_conn addr 1;
        }
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `limit_conn` | limit_conn zone number ; | — | http , server , location |
| `limit_conn_zone` | limit_conn_zone key zone = name : size ; | — | http |
| `limit_zone` | limit_zone name $variable size ; | — | http |

## Directive Details

### `limit_conn`

**Syntax**: `limit_conn zone number ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets the shared memory zone and the maximum allowed number of connections for a given key value. When this limit is exceeded, the server will return the error in reply to a request. For example, the directives limit_conn_zone $binary_remote_addr zone=addr:10m; server { location /download/ { limit_conn addr 1; } allow only one connection per an IP address at a time.

### `limit_conn_zone`

**Syntax**: `limit_conn_zone key zone = name : size ;`  
**Default**: `—`  
**Context**: `http`  

Sets parameters for a shared memory zone that will keep states for various keys. In particular, the state includes the current number of connections. The key can contain text, variables, and their combination. Requests with an empty key value are not accounted. Usage example: limit_conn_zone $binary_remote_addr zone=addr:10m; Here, a client IP address serves as a key. Note that instead of $remote_addr , the $binary_remote_addr variable is used here. The $remote_addr variable’s size can vary from 7 to 15 bytes. The stored state occupies either 32 or 64 bytes of memory on 32-bit platforms and always 64 bytes on 64-bit platforms. The $binary_remote_addr variable’s size is always 4 bytes for IPv4 addresses or 16 bytes for IPv6 addresses. The stored state always occupies 32 or 64 bytes on 32-bit platforms and 64 bytes on 64-bit platforms. One megabyte zone can keep about 32 thousand 32-byte states or about 16 thousand 64-byte states. If the zone storage is exhausted, the server will return the error to all further requests.

### `limit_zone`

**Syntax**: `limit_zone name $variable size ;`  
**Default**: `—`  
**Context**: `http`  

This directive was made obsolete in version 1.1.8 and was removed in version 1.7.6. An equivalent limit_conn_zone directive with a changed syntax should be used instead:
