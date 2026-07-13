# Module ngx_stream_proxy_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_proxy_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives proxy_bind proxy_bind_dynamic proxy_buffer_size proxy_connect_timeout proxy_download_rate proxy_half_close proxy_next_upstream proxy_next_upstream_timeout proxy_next_upstream_tries proxy_pass proxy_protocol proxy_requests proxy_responses proxy_session_drop proxy_socket_keepalive proxy_ssl proxy_ssl_alpn proxy_ssl_certificate proxy_ssl_certificate_cache proxy_ssl_certificate_key proxy_ssl_ciphers proxy_ssl_conf_command proxy_ssl_crl proxy_ssl_key_log proxy_ssl_name proxy_ssl_password_file proxy_ssl_protocols proxy_ssl_server_name proxy_ssl_session_reuse proxy_ssl_trusted_certificate proxy_ssl_verify proxy_ssl_verify_depth proxy_timeout proxy_upload_rate The ngx_stream_proxy_module module (1.9.0) allows proxying data streams over TCP, UDP (1.9.13), and UNIX-d

## Example Configuration

```nginx
server {
    listen 127.0.0.1:12345;
    proxy_pass 127.0.0.1:8080;
}

server {
    listen 12345;
    proxy_connect_timeout 1s;
    proxy_timeout 1m;
    proxy_pass example.com:12345;
}

server {
    listen 53 udp reuseport;
    proxy_timeout 20s;
    proxy_pass dns.example.com:53;
}

server {
    listen [::1]:12345;
    proxy_pass unix:/tmp/stream.socket;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `proxy_connect_timeout` | proxy_connect_timeout time ; | proxy_connect_timeout 60s; | stream , server |
| `proxy_next_upstream` | proxy_next_upstream on \| off ; | proxy_next_upstream on; | stream , server |
| `proxy_next_upstream_timeout` | proxy_next_upstream_timeout time ; | proxy_next_upstream_timeout 0; | stream , server |
| `proxy_next_upstream_tries` | proxy_next_upstream_tries number ; | proxy_next_upstream_tries 0; | stream , server |
| `proxy_pass` | proxy_pass address ; | — | server |
| `proxy_ssl` | proxy_ssl on \| off ; | proxy_ssl off; | stream , server |
| `proxy_ssl_certificate` | proxy_ssl_certificate file ; | — | stream , server |
| `proxy_ssl_certificate_key` | proxy_ssl_certificate_key file ; | — | stream , server |
| `proxy_ssl_ciphers` | proxy_ssl_ciphers ciphers ; | proxy_ssl_ciphers DEFAULT; | stream , server |
| `proxy_ssl_crl` | proxy_ssl_crl file ; | — | stream , server |
| `proxy_ssl_name` | proxy_ssl_name name ; | proxy_ssl_name host from proxy_pass; | stream , server |
| `proxy_ssl_password_file` | proxy_ssl_password_file file ; | — | stream , server |
| `proxy_ssl_protocols` | proxy_ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ]; | proxy_ssl_protocols TLSv1.2 TLSv1.3; | stream , server |
| `proxy_ssl_server_name` | proxy_ssl_server_name on \| off ; | proxy_ssl_server_name off; | stream , server |
| `proxy_ssl_session_reuse` | proxy_ssl_session_reuse on \| off ; | proxy_ssl_session_reuse on; | stream , server |
| `proxy_ssl_trusted_certificate` | proxy_ssl_trusted_certificate file ; | — | stream , server |
| `proxy_ssl_verify` | proxy_ssl_verify on \| off ; | proxy_ssl_verify off; | stream , server |
| `proxy_ssl_verify_depth` | proxy_ssl_verify_depth number ; | proxy_ssl_verify_depth 1; | stream , server |
| `proxy_timeout` | proxy_timeout timeout ; | proxy_timeout 10m; | stream , server |

## Directive Details

### `proxy_connect_timeout`

**Syntax**: `proxy_connect_timeout time ;`  
**Default**: `proxy_connect_timeout 60s;`  
**Context**: `stream , server`  

Defines a timeout for establishing a connection with a proxied server.

### `proxy_next_upstream`

**Syntax**: `proxy_next_upstream on | off ;`  
**Default**: `proxy_next_upstream on;`  
**Context**: `stream , server`  

When a connection to the proxied server cannot be established, determines whether a client connection will be passed to the next server. Passing a connection to the next server can be limited by the number of tries and by time .

### `proxy_next_upstream_timeout`

**Syntax**: `proxy_next_upstream_timeout time ;`  
**Default**: `proxy_next_upstream_timeout 0;`  
**Context**: `stream , server`  

Limits the time allowed to pass a connection to the next server . The 0 value turns off this limitation.

### `proxy_next_upstream_tries`

**Syntax**: `proxy_next_upstream_tries number ;`  
**Default**: `proxy_next_upstream_tries 0;`  
**Context**: `stream , server`  

Limits the number of possible tries for passing a connection to the next server . The 0 value turns off this limitation.

### `proxy_pass`

**Syntax**: `proxy_pass address ;`  
**Default**: `—`  
**Context**: `server`  

Sets the address of a proxied server. The address can be specified as a domain name or IP address, and a port: proxy_pass localhost:12345; or as a UNIX-domain socket path: proxy_pass unix:/tmp/stream.socket;

### `proxy_ssl`

**Syntax**: `proxy_ssl on | off ;`  
**Default**: `proxy_ssl off;`  
**Context**: `stream , server`  

Enables the SSL/TLS protocol for connections to a proxied server.

### `proxy_ssl_certificate`

**Syntax**: `proxy_ssl_certificate file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with the certificate in the PEM format used for authentication to a proxied server.

### `proxy_ssl_certificate_key`

**Syntax**: `proxy_ssl_certificate_key file ;`  
**Default**: `—`  
**Context**: `stream , server`  

The value store : scheme : id can be specified instead of the file (1.29.0), which is used to load a secret key with a specified id and OpenSSL provider registered URI scheme , such as pkcs11 . Specifies a file with the secret key in the PEM format used for authentication to a proxied server.

### `proxy_ssl_ciphers`

**Syntax**: `proxy_ssl_ciphers ciphers ;`  
**Default**: `proxy_ssl_ciphers DEFAULT;`  
**Context**: `stream , server`  

Specifies the enabled ciphers for connections to a proxied server. The ciphers are specified in the format understood by the OpenSSL library. The full list can be viewed using the “ openssl ciphers ” command.

### `proxy_ssl_crl`

**Syntax**: `proxy_ssl_crl file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with revoked certificates (CRL) in the PEM format used to verify the certificate of the proxied server. When using intermediate certificates, their CRLs should be specified in the same file.

### `proxy_ssl_name`

**Syntax**: `proxy_ssl_name name ;`  
**Default**: `proxy_ssl_name host from proxy_pass;`  
**Context**: `stream , server`  

Allows overriding the server name used to verify the certificate of the proxied server and to be passed through SNI when establishing a connection with the proxied server. The server name can also be specified using variables (1.11.3). By default, the host part of the proxy_pass address is used.

### `proxy_ssl_password_file`

**Syntax**: `proxy_ssl_password_file file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with passphrases for secret keys where each passphrase is specified on a separate line. Passphrases are tried in turn when loading the key.

### `proxy_ssl_protocols`

**Syntax**: `proxy_ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ];`  
**Default**: `proxy_ssl_protocols TLSv1.2 TLSv1.3;`  
**Context**: `stream , server`  

Enables the specified protocols for connections to a proxied server.

### `proxy_ssl_server_name`

**Syntax**: `proxy_ssl_server_name on | off ;`  
**Default**: `proxy_ssl_server_name off;`  
**Context**: `stream , server`  

Enables or disables passing of the server name through TLS Server Name Indication extension (SNI, RFC 6066) when establishing a connection with the proxied server.

### `proxy_ssl_session_reuse`

**Syntax**: `proxy_ssl_session_reuse on | off ;`  
**Default**: `proxy_ssl_session_reuse on;`  
**Context**: `stream , server`  

Determines whether SSL sessions can be reused when working with the proxied server. If the errors “ digest check failed ” appear in the logs, try disabling session reuse.

### `proxy_ssl_trusted_certificate`

**Syntax**: `proxy_ssl_trusted_certificate file ;`  
**Default**: `—`  
**Context**: `stream , server`  

Specifies a file with trusted CA certificates in the PEM format used to verify the certificate of the proxied server.

### `proxy_ssl_verify`

**Syntax**: `proxy_ssl_verify on | off ;`  
**Default**: `proxy_ssl_verify off;`  
**Context**: `stream , server`  

Enables or disables verification of the proxied server certificate.

### `proxy_ssl_verify_depth`

**Syntax**: `proxy_ssl_verify_depth number ;`  
**Default**: `proxy_ssl_verify_depth 1;`  
**Context**: `stream , server`  

Sets the verification depth in the proxied server certificates chain.

### `proxy_timeout`

**Syntax**: `proxy_timeout timeout ;`  
**Default**: `proxy_timeout 10m;`  
**Context**: `stream , server`  

Sets the timeout between two successive read or write operations on client or proxied server connections. If no data is transmitted within this time, the connection is closed.
