# Module ngx_stream_upstream_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_upstream_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives upstream server zone state hash least_conn least_time random resolver resolver_timeout Embedded Variables The ngx_stream_upstream_module module (1.9.0) is used to define groups of servers that can be referenced by the proxy_pass directive.

## Example Configuration

```nginx
upstream backend {
    hash $remote_addr consistent;

    server backend1.example.com:12345  weight=5;
    server backend2.example.com:12345;
    server unix:/tmp/backend3;

    server backup1.example.com:12345   backup;
    server backup2.example.com:12345   backup;
}

server {
    listen 12346;
    proxy_pass backend;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `upstream` | upstream name { ... } | — | stream |
| `server` | server address [ parameters ]; | — | upstream |
| `zone` | zone name [ size ]; | — | upstream |
| `hash` | hash key [ consistent ]; | — | upstream |
| `least_conn` | least_conn ; | — | upstream |

## Directive Details

### `upstream`

**Syntax**: `upstream name { ... }`  
**Default**: `—`  
**Context**: `stream`  

Defines a group of servers. Servers can listen on different ports. In addition, servers listening on TCP and UNIX-domain sockets can be mixed. Example: upstream backend { server backend1.example.com:12345 weight=5; server 127.0.0.1:12345 max_fails=3 fail_timeout=30s; server unix:/tmp/backend2; server backend3.example.com:12345 resolve; server backup1.example.com:12345 backup; }

### `server`

**Syntax**: `server address [ parameters ];`  
**Default**: `—`  
**Context**: `upstream`  

Defines the address and other parameters of a server. The address can be specified as a domain name or IP address with an obligatory port, or as a UNIX-domain socket path specified after the “ unix: ” prefix. A domain name that resolves to several IP addresses defines multiple servers at once. The following parameters can be defined:

### `zone`

**Syntax**: `zone name [ size ];`  
**Default**: `—`  
**Context**: `upstream`  

Defines the name and size of the shared memory zone that keeps the group’s configuration and run-time state that are shared between worker processes. Several groups may share the same zone. In this case, it is enough to specify the size only once. Additionally, as part of our commercial subscription , such groups allow changing the group membership or modifying the settings of a particular server without the need of restarting nginx. The configuration is accessible via the API module (1.13.3).

### `hash`

**Syntax**: `hash key [ consistent ];`  
**Default**: `—`  
**Context**: `upstream`  

Specifies a load balancing method for a server group where the client-server mapping is based on the hashed key value. The key can contain text, variables, and their combinations (1.11.2). Usage example: hash $remote_addr; Note that adding or removing a server from the group may result in remapping most of the keys to different servers. The method is compatible with the Cache::Memcached Perl library. If the consistent parameter is specified, the ketama consistent hashing method will be used instead. The method ensures that only a few keys will be remapped to different servers when a server is added to or removed from the group. This helps to achieve a higher cache hit ratio for caching servers. The method is compatible with the Cache::Memcached::Fast Perl library with the ketama_points parameter set to 160.

### `least_conn`

**Syntax**: `least_conn ;`  
**Default**: `—`  
**Context**: `upstream`  

Specifies that a group should use a load balancing method where a connection is passed to the server with the least number of active connections, taking into account weights of servers. If there are several such servers, they are tried in turn using a weighted round-robin balancing method.
