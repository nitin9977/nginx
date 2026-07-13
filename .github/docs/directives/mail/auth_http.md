# Module ngx_mail_auth_http_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_auth_http_module.html  

## Overview

Directives auth_http auth_http_header auth_http_pass_client_cert auth_http_timeout Protocol

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `auth_http` | auth_http URL ; | — | mail , server |
| `auth_http_header` | auth_http_header header value ; | — | mail , server |
| `auth_http_timeout` | auth_http_timeout time ; | auth_http_timeout 60s; | mail , server |

## Directive Details

### `auth_http`

**Syntax**: `auth_http URL ;`  
**Default**: `—`  
**Context**: `mail , server`  

Sets the URL of the HTTP authentication server. The protocol is described below .

### `auth_http_header`

**Syntax**: `auth_http_header header value ;`  
**Default**: `—`  
**Context**: `mail , server`  

Appends the specified header to requests sent to the authentication server. This header can be used as the shared secret to verify that the request comes from nginx. For example: auth_http_header X-Auth-Key "secret_string";

### `auth_http_timeout`

**Syntax**: `auth_http_timeout time ;`  
**Default**: `auth_http_timeout 60s;`  
**Context**: `mail , server`  

Sets the timeout for communication with the authentication server.
