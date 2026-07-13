# Module ngx_stream_core_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_core_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives listen preread_buffer_size preread_timeout proxy_protocol_timeout resolver resolver_timeout server server_name server_names_hash_bucket_size server_names_hash_max_size stream tcp_nodelay variables_hash_bucket_size variables_hash_max_size Embedded Variables The ngx_stream_core_module module is available since version 1.9.0. This module is not built by default, it should be enabled with the --with-stream configuration parameter.

## Example Configuration

```nginx
worker_processes auto;

error_log /var/log/nginx/error.log info;

events {
    worker_connections  1024;
}

stream {
    upstream backend {
        hash $remote_addr consistent;

        server backend1.example.com:12345 weight=5;
        server 127.0.0.1:12345            max_fails=3 fail_timeout=30s;
        server unix:/tmp/backend3;
    }

    upstream dns {
       server 192.168.0.1:53535;
       server dns.example.com:53;
    }

    server {
        listen 12345;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass backend;
    }

    server {
        listen 127.0.0.1:53 udp reuseport;
        proxy_timeout 20s;
        proxy_pass dns;
    }

    server {
        listen [::1]:12345;
        proxy_pass unix:/tmp/stream.socket;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `listen` | listen address : port [ default_server ] [ ssl ] [ udp ] [ proxy_protocol ] [ setfib = number ] [ fastopen = number ] [  | — | server |
| `server` | server { ... } | — | stream |
| `stream` | stream { ... } | — | main |

## Directive Details

### `listen`

**Syntax**: `listen address : port [ default_server ] [ ssl ] [ udp ] [ proxy_protocol ] [ setfib = number ] [ fastopen = number ] [ backlog = number ] [ rcvbuf = size ] [ sndbuf = size ] [ accept_filter = filter ] [ deferred ] [ bind ] [ ipv6only = on | off ] [ reuseport ] [ multipath ] [ so_keepalive = on | off |[ keepidle ]:[ keepintvl ]:[ keepcnt ]];`  
**Default**: `—`  
**Context**: `server`  

Sets the address and port for the socket on which the server will accept connections. It is possible to specify just the port. The address can also be a hostname, for example: listen 127.0.0.1:12345; listen *:12345; listen 12345; # same as *:12345 listen localhost:12345; IPv6 addresses are specified in square brackets: listen [::1]:12345; listen [::]:12345; UNIX-domain sockets are specified with the “ unix: ” prefix:

### `server`

**Syntax**: `server { ... }`  
**Default**: `—`  
**Context**: `stream`  

Sets the configuration for a virtual server. There is no clear separation between IP-based (based on the IP address) and name-based (based on the TLS Server Name Indication extension (SNI, RFC 6066)) (1.25.5) virtual servers. Instead, the listen directives describe all addresses and ports that should accept connections for the server, and the server_name directive lists all server names.

### `stream`

**Syntax**: `stream { ... }`  
**Default**: `—`  
**Context**: `main`  

Provides the configuration file context in which the stream server directives are specified.
