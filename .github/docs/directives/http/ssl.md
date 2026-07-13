# Module ngx_http_ssl_module

**Source**: https://nginx.org/en/docs/http/ngx_http_ssl_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives ssl ssl_buffer_size ssl_certificate ssl_certificate_cache ssl_certificate_compression ssl_certificate_key ssl_ciphers ssl_client_certificate ssl_conf_command ssl_crl ssl_dhparam ssl_early_data ssl_ecdh_curve ssl_ech_file ssl_key_log ssl_ocsp ssl_ocsp_cache ssl_ocsp_responder ssl_password_file ssl_prefer_server_ciphers ssl_protocols ssl_reject_handshake ssl_session_cache ssl_session_ticket_key ssl_session_tickets ssl_session_timeout ssl_stapling ssl_stapling_file ssl_stapling_responder ssl_stapling_verify ssl_trusted_certificate ssl_verify_client ssl_verify_depth Error Processing Embedded Variables The ngx_http_ssl_module module provides the necessary support for HTTPS. This module is not built by default, it should be enabled with the --with-http_ssl_module

## Example Configuration

```nginx
worker_processes auto;

http {

    ...

    server {
        listen              443 ssl;
        keepalive_timeout   70;

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
| `ssl` | ssl on \| off ; | ssl off; | http , server |
| `ssl_certificate` | ssl_certificate file ; | — | http , server |
| `ssl_certificate_key` | ssl_certificate_key file ; | — | http , server |
| `ssl_ciphers` | ssl_ciphers ciphers ; | ssl_ciphers HIGH:!aNULL:!MD5; | http , server |
| `ssl_client_certificate` | ssl_client_certificate file ; | — | http , server |
| `ssl_prefer_server_ciphers` | ssl_prefer_server_ciphers on \| off ; | ssl_prefer_server_ciphers off; | http , server |
| `ssl_protocols` | ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ]; | ssl_protocols TLSv1.2 TLSv1.3; | http , server |
| `ssl_session_cache` | ssl_session_cache off \| none \| [ builtin [: size ]] [ shared : name : size ]; | ssl_session_cache none; | http , server |
| `ssl_session_timeout` | ssl_session_timeout time ; | ssl_session_timeout 5m; | http , server |
| `ssl_verify_client` | ssl_verify_client on \| off \| optional \| optional_no_ca ; | ssl_verify_client off; | http , server |
| `ssl_verify_depth` | ssl_verify_depth number ; | ssl_verify_depth 1; | http , server |

## Directive Details

### `ssl`

**Syntax**: `ssl on | off ;`  
**Default**: `ssl off;`  
**Context**: `http , server`  

This directive was made obsolete in version 1.15.0 and was removed in version 1.25.1. The ssl parameter of the listen directive should be used instead.

### `ssl_certificate`

**Syntax**: `ssl_certificate file ;`  
**Default**: `—`  
**Context**: `http , server`  

Specifies a file with the certificate in the PEM format for the given virtual server. If intermediate certificates should be specified in addition to a primary certificate, they should be specified in the same file in the following order: the primary certificate comes first, then the intermediate certificates. A secret key in the PEM format may be placed in the same file. Since version 1.11.0, this directive can be specified multiple times to load certificates of different types, for example, RSA and ECDSA: server { listen 443 ssl; server_name example.com; ssl_certificate example.com.rsa.crt; ssl_certificate_key example.com.rsa.key; ssl_certificate example.com.ecdsa.crt; ssl_certificate_key example.com.ecdsa.key; ... }

### `ssl_certificate_key`

**Syntax**: `ssl_certificate_key file ;`  
**Default**: `—`  
**Context**: `http , server`  

Specifies a file with the secret key in the PEM format for the given virtual server. The value engine : name : id can be specified instead of the file (1.7.9), which loads a secret key with a specified id from the OpenSSL engine name . The value store : scheme : id can be specified instead of the file (1.29.0), which is used to load a secret key with a specified id and OpenSSL provider registered URI scheme , such as pkcs11 .

### `ssl_ciphers`

**Syntax**: `ssl_ciphers ciphers ;`  
**Default**: `ssl_ciphers HIGH:!aNULL:!MD5;`  
**Context**: `http , server`  

Specifies the enabled ciphers. The ciphers are specified in the format understood by the OpenSSL library, for example: ssl_ciphers ALL:!aNULL:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP; The full list can be viewed using the “ openssl ciphers ” command.

### `ssl_client_certificate`

**Syntax**: `ssl_client_certificate file ;`  
**Default**: `—`  
**Context**: `http , server`  

Specifies a file with trusted CA certificates in the PEM format used to verify client certificates and OCSP responses if ssl_stapling is enabled. The list of certificates will be sent to clients. If this is not desired, the ssl_trusted_certificate directive can be used.

### `ssl_prefer_server_ciphers`

**Syntax**: `ssl_prefer_server_ciphers on | off ;`  
**Default**: `ssl_prefer_server_ciphers off;`  
**Context**: `http , server`  

Specifies that server ciphers should be preferred over client ciphers when the SSLv3 and TLS protocols are used.

### `ssl_protocols`

**Syntax**: `ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ];`  
**Default**: `ssl_protocols TLSv1.2 TLSv1.3;`  
**Context**: `http , server`  

Enables the specified protocols. If the directive is specified on the server level, the value from the default server can be used. Details are provided in the “ Virtual server selection ” section.

### `ssl_session_cache`

**Syntax**: `ssl_session_cache off | none | [ builtin [: size ]] [ shared : name : size ];`  
**Default**: `ssl_session_cache none;`  
**Context**: `http , server`  

Sets the types and sizes of caches that store session parameters. A cache can be of any of the following types: Both cache types can be used simultaneously, for example:

### `ssl_session_timeout`

**Syntax**: `ssl_session_timeout time ;`  
**Default**: `ssl_session_timeout 5m;`  
**Context**: `http , server`  

Specifies a time during which a client may reuse the session parameters.

### `ssl_verify_client`

**Syntax**: `ssl_verify_client on | off | optional | optional_no_ca ;`  
**Default**: `ssl_verify_client off;`  
**Context**: `http , server`  

Enables verification of client certificates. The verification result is stored in the $ssl_client_verify variable. The optional parameter (0.8.7+) requests the client certificate and verifies it if the certificate is present. The optional_no_ca parameter (1.3.8, 1.2.5) requests the client certificate but does not require it to be signed by a trusted CA certificate. This is intended for the use in cases when a service that is external to nginx performs the actual certificate verification. The contents of the certificate is accessible through the $ssl_client_cert variable.

### `ssl_verify_depth`

**Syntax**: `ssl_verify_depth number ;`  
**Default**: `ssl_verify_depth 1;`  
**Context**: `http , server`  

Sets the verification depth in the client certificates chain.
