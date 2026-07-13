# Module ngx_mail_core_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_core_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives listen mail max_errors protocol resolver resolver_timeout server server_name timeout This module is not built by default, it should be enabled with the --with-mail configuration parameter.

## Example Configuration

```nginx
worker_processes auto;

error_log /var/log/nginx/error.log info;

events {
    worker_connections  1024;
}

mail {
    server_name       mail.example.com;
    auth_http         localhost:9000/cgi-bin/nginxauth.cgi;

    imap_capabilities IMAP4rev1 UIDPLUS IDLE LITERAL+ QUOTA;

    pop3_auth         plain apop cram-md5;
    pop3_capabilities LAST TOP USER PIPELINING UIDL;

    smtp_auth         login plain cram-md5;
    smtp_capabilities "SIZE 10485760" ENHANCEDSTATUSCODES 8BITMIME DSN;
    xclient           off;

    server {
        listen   25;
        protocol smtp;
    }
    server {
        listen   110;
        protocol pop3;
        proxy_pass_error_message on;
    }
    server {
        listen   143;
        protocol imap;
    }
    server {
        listen   587;
        protocol smtp;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `listen` | listen address : port [ ssl ] [ proxy_protocol ] [ backlog = number ] [ rcvbuf = size ] [ sndbuf = size ] [ bind ] [ ipv | — | server |
| `mail` | mail { ... } | — | main |
| `protocol` | protocol imap \| pop3 \| smtp ; | — | server |
| `resolver` | resolver address ... [ valid = time ] [ ipv4 = on \| off ] [ ipv6 = on \| off ] [ status_zone = zone ]; resolver off ; | resolver off; | mail , server |
| `resolver_timeout` | resolver_timeout time ; | resolver_timeout 30s; | mail , server |
| `server` | server { ... } | — | mail |
| `server_name` | server_name name ; | server_name hostname; | mail , server |
| `timeout` | timeout time ; | timeout 60s; | mail , server |

## Directive Details

### `listen`

**Syntax**: `listen address : port [ ssl ] [ proxy_protocol ] [ backlog = number ] [ rcvbuf = size ] [ sndbuf = size ] [ bind ] [ ipv6only = on | off ] [ multipath ] [ so_keepalive = on | off |[ keepidle ]:[ keepintvl ]:[ keepcnt ]];`  
**Default**: `—`  
**Context**: `server`  

Sets the address and port for the socket on which the server will accept requests. It is possible to specify just the port. The address can also be a hostname, for example: listen 127.0.0.1:110; listen *:110; listen 110; # same as *:110 listen localhost:110; IPv6 addresses (0.7.58) are specified in square brackets: listen [::1]:110; listen [::]:110; UNIX-domain sockets (1.3.5) are specified with the “ unix: ” prefix:

### `mail`

**Syntax**: `mail { ... }`  
**Default**: `—`  
**Context**: `main`  

Provides the configuration file context in which the mail server directives are specified.

### `protocol`

**Syntax**: `protocol imap | pop3 | smtp ;`  
**Default**: `—`  
**Context**: `server`  

Sets the protocol for a proxied server. Supported protocols are IMAP , POP3 , and SMTP . If the directive is not set, the protocol can be detected automatically based on the well-known port specified in the listen directive:

### `resolver`

**Syntax**: `resolver address ... [ valid = time ] [ ipv4 = on | off ] [ ipv6 = on | off ] [ status_zone = zone ]; resolver off ;`  
**Default**: `resolver off;`  
**Context**: `mail , server`  

Configures name servers used to find the client’s hostname to pass it to the authentication server , and in the XCLIENT command when proxying SMTP. For example: resolver 127.0.0.1 [::1]:5353; The address can be specified as a domain name or IP address, with an optional port (1.3.1, 1.2.2). If port is not specified, the port 53 is used. Name servers are queried in a round-robin fashion.

### `resolver_timeout`

**Syntax**: `resolver_timeout time ;`  
**Default**: `resolver_timeout 30s;`  
**Context**: `mail , server`  

Sets a timeout for DNS operations, for example: resolver_timeout 5s;

### `server`

**Syntax**: `server { ... }`  
**Default**: `—`  
**Context**: `mail`  

Sets the configuration for a server.

### `server_name`

**Syntax**: `server_name name ;`  
**Default**: `server_name hostname;`  
**Context**: `mail , server`  

Sets the server name that is used: If the directive is not specified, the machine’s hostname is used.

### `timeout`

**Syntax**: `timeout time ;`  
**Default**: `timeout 60s;`  
**Context**: `mail , server`  

Sets the timeout that is used before proxying to the backend starts.
