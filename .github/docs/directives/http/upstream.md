# Module ngx_http_upstream_module

**Source**: https://nginx.org/en/docs/http/ngx_http_upstream_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives upstream server zone state hash ip_hash keepalive keepalive_requests keepalive_time keepalive_timeout ntlm least_conn least_time queue random resolver resolver_timeout sticky Embedded Variables The ngx_http_upstream_module module is used to define groups of servers that can be referenced by the proxy_pass , fastcgi_pass , uwsgi_pass , scgi_pass , memcached_pass , and grpc_pass directives.

## Example Configuration

```nginx
upstream backend {
    server backend1.example.com       weight=5;
    server backend2.example.com:8080;
    server unix:/tmp/backend3;

    server backup1.example.com:8080   backup;
    server backup2.example.com:8080   backup;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `upstream` | upstream name { ... } | — | http |
| `server` | server address [ parameters ]; | — | upstream |
| `ip_hash` | ip_hash ; | — | upstream |

## Directive Details

### `upstream`

**Syntax**: `upstream name { ... }`  
**Default**: `—`  
**Context**: `http`  

Defines a group of servers. Servers can listen on different ports. In addition, servers listening on TCP and UNIX-domain sockets can be mixed. Example: upstream backend { server backend1.example.com weight=5; server 127.0.0.1:8080 max_fails=3 fail_timeout=30s; server unix:/tmp/backend3; server backup1.example.com backup; }

### `server`

**Syntax**: `server address [ parameters ];`  
**Default**: `—`  
**Context**: `upstream`  

Defines the address and other parameters of a server. The address can be specified as a domain name or IP address, with an optional port, or as a UNIX-domain socket path specified after the “ unix: ” prefix. If a port is not specified, the port 80 is used. A domain name that resolves to several IP addresses defines multiple servers at once. The following parameters can be defined:

### `ip_hash`

**Syntax**: `ip_hash ;`  
**Default**: `—`  
**Context**: `upstream`  

Specifies that a group should use a load balancing method where requests are distributed between servers based on client IP addresses. The first three octets of the client IPv4 address, or the entire IPv6 address, are used as a hashing key. The method ensures that requests from the same client will always be passed to the same server except when this server is unavailable. In the latter case client requests will be passed to another server. Most probably, it will always be the same server as well. If one of the servers needs to be temporarily removed, it should be marked with the down parameter in order to preserve the current hashing of client IP addresses.
