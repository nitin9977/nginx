# Module ngx_stream_limit_conn_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_limit_conn_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives limit_conn limit_conn_dry_run limit_conn_log_level limit_conn_zone Embedded Variables The ngx_stream_limit_conn_module module (1.9.3) is used to limit the number of connections per the defined key, in particular, the number of connections from a single IP address.

## Example Configuration

```nginx
stream {
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    ...

    server {

        ...

        limit_conn           addr 1;
        limit_conn_log_level error;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `limit_conn` | limit_conn zone number ; | — | stream , server |
| `limit_conn_log_level` | limit_conn_log_level info \| notice \| warn \| error ; | limit_conn_log_level error; | stream , server |
| `limit_conn_zone` | limit_conn_zone key zone = name : size ; | — | stream |

## Directive Details

### `limit_conn`

**Syntax**: `limit_conn zone number ;`  
**Default**: `—`  
**Context**: `stream , server`  

Sets the shared memory zone and the maximum allowed number of connections for a given key value. When this limit is exceeded, the server will close the connection. For example, the directives limit_conn_zone $binary_remote_addr zone=addr:10m; server { ... limit_conn addr 1; } allow only one connection per an IP address at a time. When several limit_conn directives are specified, any configured limit will apply.

### `limit_conn_log_level`

**Syntax**: `limit_conn_log_level info | notice | warn | error ;`  
**Default**: `limit_conn_log_level error;`  
**Context**: `stream , server`  

Sets the desired logging level for cases when the server limits the number of connections.

### `limit_conn_zone`

**Syntax**: `limit_conn_zone key zone = name : size ;`  
**Default**: `—`  
**Context**: `stream`  

Sets parameters for a shared memory zone that will keep states for various keys. In particular, the state includes the current number of connections. The key can contain text, variables, and their combinations (1.11.2). Connections with an empty key value are not accounted. Usage example: limit_conn_zone $binary_remote_addr zone=addr:10m; Here, the key is a client IP address set by the $binary_remote_addr variable. The size of $binary_remote_addr is 4 bytes for IPv4 addresses or 16 bytes for IPv6 addresses. The stored state always occupies 32 or 64 bytes on 32-bit platforms and 64 bytes on 64-bit platforms. One megabyte zone can keep about 32 thousand 32-byte states or about 16 thousand 64-byte states. If the zone storage is exhausted, the server will close the connection.
