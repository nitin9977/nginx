# Module ngx_mail_ssl_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_ssl_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives ssl ssl_certificate ssl_certificate_compression ssl_certificate_key ssl_ciphers ssl_client_certificate ssl_conf_command ssl_crl ssl_dhparam ssl_ecdh_curve ssl_password_file ssl_prefer_server_ciphers ssl_protocols ssl_session_cache ssl_session_ticket_key ssl_session_tickets ssl_session_timeout ssl_trusted_certificate ssl_verify_client ssl_verify_depth starttls The ngx_mail_ssl_module module provides the necessary support for a mail proxy server to work with the SSL/TLS protocol. This module is not built by default, it should be enabled with the --with-mail_ssl_module configuration parameter. This module requires the OpenSSL library.

## Example Configuration

```nginx
worker_processes auto;

mail {

    ...

    server {
        listen              993 ssl;

        ssl_protocols       TLSv1.2 TLSv1.3;
        ssl_ciphers         AES128-SHA:AES256-SHA:RC4-SHA:DES-CBC3-SHA:RC4-MD5;
        ssl_certificate     /usr/local/nginx/conf/cert.pem;
        ssl_certificate_key /usr/local/nginx/conf/cert.key;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 10m;

        ...
    }
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `ssl` | ssl on \| off ; | ssl off; | mail , server |
| `ssl_certificate` | ssl_certificate file ; | — | mail , server |
| `ssl_certificate_key` | ssl_certificate_key file ; | — | mail , server |
| `ssl_ciphers` | ssl_ciphers ciphers ; | ssl_ciphers HIGH:!aNULL:!MD5; | mail , server |
| `ssl_prefer_server_ciphers` | ssl_prefer_server_ciphers on \| off ; | ssl_prefer_server_ciphers off; | mail , server |
| `ssl_protocols` | ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ]; | ssl_protocols TLSv1.2 TLSv1.3; | mail , server |
| `ssl_session_cache` | ssl_session_cache off \| none \| [ builtin [: size ]] [ shared : name : size ]; | ssl_session_cache none; | mail , server |
| `ssl_session_timeout` | ssl_session_timeout time ; | ssl_session_timeout 5m; | mail , server |
| `starttls` | starttls on \| off \| only ; | starttls off; | mail , server |

## Directive Details

### `ssl`

**Syntax**: `ssl on | off ;`  
**Default**: `ssl off;`  
**Context**: `mail , server`  

This directive was made obsolete in version 1.15.0 and was removed in version 1.25.1. The ssl parameter of the listen directive should be used instead.

### `ssl_certificate`

**Syntax**: `ssl_certificate file ;`  
**Default**: `—`  
**Context**: `mail , server`  

Specifies a file with the certificate in the PEM format for the given server. If intermediate certificates should be specified in addition to a primary certificate, they should be specified in the same file in the following order: the primary certificate comes first, then the intermediate certificates. A secret key in the PEM format may be placed in the same file. Since version 1.11.0, this directive can be specified multiple times to load certificates of different types, for example, RSA and ECDSA: server { listen 993 ssl; ssl_certificate example.com.rsa.crt; ssl_certificate_key example.com.rsa.key; ssl_certificate example.com.ecdsa.crt; ssl_certificate_key example.com.ecdsa.key; ... }

### `ssl_certificate_key`

**Syntax**: `ssl_certificate_key file ;`  
**Default**: `—`  
**Context**: `mail , server`  

Specifies a file with the secret key in the PEM format for the given server. The value engine : name : id can be specified instead of the file (1.7.9), which loads a secret key with a specified id from the OpenSSL engine name . The value store : scheme : id can be specified instead of the file (1.29.0), which is used to load a secret key with a specified id and OpenSSL provider registered URI scheme , such as pkcs11 .

### `ssl_ciphers`

**Syntax**: `ssl_ciphers ciphers ;`  
**Default**: `ssl_ciphers HIGH:!aNULL:!MD5;`  
**Context**: `mail , server`  

Specifies the enabled ciphers. The ciphers are specified in the format understood by the OpenSSL library, for example: ssl_ciphers ALL:!aNULL:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP; The full list can be viewed using the “ openssl ciphers ” command.

### `ssl_prefer_server_ciphers`

**Syntax**: `ssl_prefer_server_ciphers on | off ;`  
**Default**: `ssl_prefer_server_ciphers off;`  
**Context**: `mail , server`  

Specifies that server ciphers should be preferred over client ciphers when the SSLv3 and TLS protocols are used.

### `ssl_protocols`

**Syntax**: `ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ];`  
**Default**: `ssl_protocols TLSv1.2 TLSv1.3;`  
**Context**: `mail , server`  

Enables the specified protocols.

### `ssl_session_cache`

**Syntax**: `ssl_session_cache off | none | [ builtin [: size ]] [ shared : name : size ];`  
**Default**: `ssl_session_cache none;`  
**Context**: `mail , server`  

Sets the types and sizes of caches that store session parameters. A cache can be of any of the following types: Both cache types can be used simultaneously, for example:

### `ssl_session_timeout`

**Syntax**: `ssl_session_timeout time ;`  
**Default**: `ssl_session_timeout 5m;`  
**Context**: `mail , server`  

Specifies a time during which a client may reuse the session parameters.

### `starttls`

**Syntax**: `starttls on | off | only ;`  
**Default**: `starttls off;`  
**Context**: `mail , server`  

