# Module ngx_stream_ssl_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives ssl_alpn ssl_certificate ssl_certificate_cache ssl_certificate_compression ssl_certificate_key ssl_ciphers ssl_client_certificate ssl_conf_command ssl_crl ssl_dhparam ssl_ecdh_curve ssl_ech_file ssl_handshake_timeout ssl_key_log ssl_ocsp ssl_ocsp_cache ssl_ocsp_responder ssl_password_file ssl_prefer_server_ciphers ssl_protocols ssl_reject_handshake ssl_session_cache ssl_session_ticket_key ssl_session_tickets ssl_session_timeout ssl_stapling ssl_stapling_file ssl_stapling_responder ssl_stapling_verify ssl_trusted_certificate ssl_verify_client ssl_verify_depth Embedded Variables The ngx_stream_ssl_module module (1.9.0) provides the necessary support for a stream proxy server to work with the SSL/TLS protocol. This module is not built by default, it should be 

## Example Configuration

```nginx
worker_processes auto;

stream {

    ...

    server {
        listen              12345 ssl;

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
| `ssl_certificate` | ssl_certificate file ; | — | stream , server |
| `ssl_certificate_key` | ssl_certificate_key file ; | — | stream , server |
| `ssl_ciphers` | ssl_ciphers ciphers ; | ssl_ciphers HIGH:!aNULL:!MD5; | stream , server |
| `ssl_dhparam` | ssl_dhparam file ; | — | stream , server |
| `ssl_ecdh_curve` | ssl_ecdh_curve curve ; | ssl_ecdh_curve auto; | stream , server |
| `ssl_handshake_timeout` | ssl_handshake_timeout time ; | ssl_handshake_timeout 60s; | stream , server |
| `ssl_password_file` | ssl_password_file file ; | — | stream , server |
| `ssl_prefer_server_ciphers` | ssl_prefer_server_ciphers on \| off ; | ssl_prefer_server_ciphers off; | stream , server |
| `ssl_protocols` | ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ]; | ssl_protocols TLSv1.2 TLSv1.3; | stream , server |
| `ssl_session_cache` | ssl_session_cache off \| none \| [ builtin [: size ]] [ shared : name : size ]; | ssl_session_cache none; | stream , server |
| `ssl_session_ticket_key` | ssl_session_ticket_key file ; | — | stream , server |
| `ssl_session_tickets` | ssl_session_tickets on \| off ; | ssl_session_tickets on; | stream , server |
| `ssl_session_timeout` | ssl_session_timeout time ; | ssl_session_timeout 5m; | stream , server |

## Directive Details

### `ssl_certificate`

**Syntax**: `ssl_certificate file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with the certificate in the PEM format for the given virtual server. If intermediate certificates should be specified in addition to a primary certificate, they should be specified in the same file in the following order: the primary certificate comes first, then the intermediate certificates. A secret key in the PEM format may be placed in the same file. Since version 1.11.0, this directive can be specified multiple times to load certificates of different types, for example, RSA and ECDSA: server { listen 12345 ssl; ssl_certificate example.com.rsa.crt; ssl_certificate_key example.com.rsa.key; ssl_certificate example.com.ecdsa.crt; ssl_certificate_key example.com.ecdsa.key; ... }

### `ssl_certificate_key`

**Syntax**: `ssl_certificate_key file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with the secret key in the PEM format for the given virtual server. The value engine : name : id can be specified instead of the file , which loads a secret key with a specified id from the OpenSSL engine name . The value store : scheme : id can be specified instead of the file (1.29.0), which is used to load a secret key with a specified id and OpenSSL provider registered URI scheme , such as pkcs11 .

### `ssl_ciphers`

**Syntax**: `ssl_ciphers ciphers ;`  
**Default**: `ssl_ciphers HIGH:!aNULL:!MD5;`  
**Context**: `stream , server`  

Specifies the enabled ciphers. The ciphers are specified in the format understood by the OpenSSL library, for example: ssl_ciphers ALL:!aNULL:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP; The full list can be viewed using the “ openssl ciphers ” command.

### `ssl_dhparam`

**Syntax**: `ssl_dhparam file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with DH parameters for DHE ciphers. By default no parameters are set, and therefore DHE ciphers will not be used.

### `ssl_ecdh_curve`

**Syntax**: `ssl_ecdh_curve curve ;`  
**Default**: `ssl_ecdh_curve auto;`  
**Context**: `stream , server`  

Specifies a curve for ECDHE ciphers. When using OpenSSL 1.0.2 or higher, it is possible to specify multiple curves (1.11.0), for example: ssl_ecdh_curve prime256v1:secp384r1;

### `ssl_handshake_timeout`

**Syntax**: `ssl_handshake_timeout time ;`  
**Default**: `ssl_handshake_timeout 60s;`  
**Context**: `stream , server`  

Specifies a timeout for the SSL handshake to complete.

### `ssl_password_file`

**Syntax**: `ssl_password_file file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with passphrases for secret keys where each passphrase is specified on a separate line. Passphrases are tried in turn when loading the key. Example: stream { ssl_password_file /etc/keys/global.pass; ... server { listen 127.0.0.1:12345; ssl_certificate_key /etc/keys/first.key; } server { listen 127.0.0.1:12346; # named pipe can also be used instead of a file ssl_password_file /etc/keys/fifo; ssl_certificate_key /etc/keys/second.key; } }

### `ssl_prefer_server_ciphers`

**Syntax**: `ssl_prefer_server_ciphers on | off ;`  
**Default**: `ssl_prefer_server_ciphers off;`  
**Context**: `stream , server`  

Specifies that server ciphers should be preferred over client ciphers when the SSLv3 and TLS protocols are used.

### `ssl_protocols`

**Syntax**: `ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ];`  
**Default**: `ssl_protocols TLSv1.2 TLSv1.3;`  
**Context**: `stream , server`  

Enables the specified protocols. If the directive is specified on the server level, the value from the default server can be used. Details are provided in the “ Virtual server selection ” section.

### `ssl_session_cache`

**Syntax**: `ssl_session_cache off | none | [ builtin [: size ]] [ shared : name : size ];`  
**Default**: `ssl_session_cache none;`  
**Context**: `stream , server`  

Sets the types and sizes of caches that store session parameters. A cache can be of any of the following types: Both cache types can be used simultaneously, for example:

### `ssl_session_ticket_key`

**Syntax**: `ssl_session_ticket_key file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Sets a file with the secret key used to encrypt and decrypt TLS session tickets. The directive is necessary if the same key has to be shared between multiple servers. By default, a randomly generated key is used. If several keys are specified, only the first key is used to encrypt TLS session tickets. This allows configuring key rotation, for example: ssl_session_ticket_key current.key; ssl_session_ticket_key previous.key;

### `ssl_session_tickets`

**Syntax**: `ssl_session_tickets on | off ;`  
**Default**: `ssl_session_tickets on;`  
**Context**: `stream , server`  

Enables or disables session resumption through TLS session tickets .

### `ssl_session_timeout`

**Syntax**: `ssl_session_timeout time ;`  
**Default**: `ssl_session_timeout 5m;`  
**Context**: `stream , server`  

Specifies a time during which a client may reuse the session parameters.
