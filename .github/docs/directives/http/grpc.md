# Module ngx_http_grpc_module

**Source**: https://nginx.org/en/docs/http/ngx_http_grpc_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives grpc_allow_upstream grpc_bind grpc_bind_dynamic grpc_buffer_size grpc_connect_timeout grpc_hide_header grpc_ignore_headers grpc_intercept_errors grpc_next_upstream grpc_next_upstream_timeout grpc_next_upstream_tries grpc_pass grpc_pass_header grpc_read_timeout grpc_request_dynamic grpc_send_timeout grpc_set_header grpc_socket_keepalive grpc_ssl_certificate grpc_ssl_certificate_cache grpc_ssl_certificate_key grpc_ssl_ciphers grpc_ssl_conf_command grpc_ssl_crl grpc_ssl_key_log grpc_ssl_name grpc_ssl_password_file grpc_ssl_protocols grpc_ssl_server_name grpc_ssl_session_reuse grpc_ssl_trusted_certificate grpc_ssl_verify grpc_ssl_verify_depth The ngx_http_grpc_module module allows passing requests to a gRPC server (1.13.10). The module requires the ngx_http_v2_

## Example Configuration

```nginx
server {
    listen 9000;

    http2 on;

    location / {
        grpc_pass 127.0.0.1:9000;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `grpc_bind` | grpc_bind address [ transparent ] \| off ; | — | http , server , location |
| `grpc_buffer_size` | grpc_buffer_size size ; | grpc_buffer_size 4k\|8k; | http , server , location |
| `grpc_connect_timeout` | grpc_connect_timeout time ; | grpc_connect_timeout 60s; | http , server , location |
| `grpc_hide_header` | grpc_hide_header field ; | — | http , server , location |
| `grpc_ignore_headers` | grpc_ignore_headers field ...; | — | http , server , location |
| `grpc_intercept_errors` | grpc_intercept_errors on \| off ; | grpc_intercept_errors off; | http , server , location |
| `grpc_next_upstream` | grpc_next_upstream error \| timeout \| denied \| invalid_header \| http_500 \| http_502 \| http_503 \| http_504 \| http_ | grpc_next_upstream error timeout; | http , server , location |
| `grpc_next_upstream_timeout` | grpc_next_upstream_timeout time ; | grpc_next_upstream_timeout 0; | http , server , location |
| `grpc_next_upstream_tries` | grpc_next_upstream_tries number ; | grpc_next_upstream_tries 0; | http , server , location |
| `grpc_pass` | grpc_pass address ; | — | location , if in location |
| `grpc_pass_header` | grpc_pass_header field ; | — | http , server , location |
| `grpc_read_timeout` | grpc_read_timeout time ; | grpc_read_timeout 60s; | http , server , location |
| `grpc_send_timeout` | grpc_send_timeout time ; | grpc_send_timeout 60s; | http , server , location |
| `grpc_set_header` | grpc_set_header field value ; | grpc_set_header Content-Length $content_length; | http , server , location |
| `grpc_ssl_certificate` | grpc_ssl_certificate file ; | — | http , server , location |
| `grpc_ssl_certificate_key` | grpc_ssl_certificate_key file ; | — | http , server , location |
| `grpc_ssl_ciphers` | grpc_ssl_ciphers ciphers ; | grpc_ssl_ciphers DEFAULT; | http , server , location |
| `grpc_ssl_crl` | grpc_ssl_crl file ; | — | http , server , location |
| `grpc_ssl_name` | grpc_ssl_name name ; | grpc_ssl_name host from grpc_pass; | http , server , location |
| `grpc_ssl_password_file` | grpc_ssl_password_file file ; | — | http , server , location |
| `grpc_ssl_protocols` | grpc_ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ]; | grpc_ssl_protocols TLSv1.2 TLSv1.3; | http , server , location |
| `grpc_ssl_server_name` | grpc_ssl_server_name on \| off ; | grpc_ssl_server_name off; | http , server , location |
| `grpc_ssl_session_reuse` | grpc_ssl_session_reuse on \| off ; | grpc_ssl_session_reuse on; | http , server , location |
| `grpc_ssl_trusted_certificate` | grpc_ssl_trusted_certificate file ; | — | http , server , location |
| `grpc_ssl_verify` | grpc_ssl_verify on \| off ; | grpc_ssl_verify off; | http , server , location |
| `grpc_ssl_verify_depth` | grpc_ssl_verify_depth number ; | grpc_ssl_verify_depth 1; | http , server , location |

## Directive Details

### `grpc_bind`

**Syntax**: `grpc_bind address [ transparent ] | off ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Makes outgoing connections to a gRPC server originate from the specified local IP address with an optional port. Parameter value can contain variables. The special value off cancels the effect of the grpc_bind directive inherited from the previous configuration level, which allows the system to auto-assign the local IP address and port.

### `grpc_buffer_size`

**Syntax**: `grpc_buffer_size size ;`  
**Default**: `grpc_buffer_size 4k|8k;`  
**Context**: `http , server , location`  

Sets the size of the buffer used for reading the response received from the gRPC server. The first part of the response usually contains a small header; if it exceeds the buffer size, the response is considered invalid . The response is passed to the client synchronously, as soon as it is received. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform. It can be made smaller, however.

### `grpc_connect_timeout`

**Syntax**: `grpc_connect_timeout time ;`  
**Default**: `grpc_connect_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for establishing a connection with a gRPC server. It should be noted that this timeout cannot usually exceed 75 seconds.

### `grpc_hide_header`

**Syntax**: `grpc_hide_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

By default, nginx does not pass the header fields “Date”, “Server”, and “X-Accel-...” from the response of a gRPC server to a client. The grpc_hide_header directive sets additional fields that will not be passed. If, on the contrary, the passing of fields needs to be permitted, the grpc_pass_header directive can be used.

### `grpc_ignore_headers`

**Syntax**: `grpc_ignore_headers field ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Disables processing of certain response header fields from the gRPC server. The following fields can be ignored: “X-Accel-Redirect” and “X-Accel-Charset”. If not disabled, processing of these header fields has the following effect:

### `grpc_intercept_errors`

**Syntax**: `grpc_intercept_errors on | off ;`  
**Default**: `grpc_intercept_errors off;`  
**Context**: `http , server , location`  

Determines whether gRPC server responses with codes greater than or equal to 300 should be passed to a client or be intercepted and redirected to nginx for processing with the error_page directive.

### `grpc_next_upstream`

**Syntax**: `grpc_next_upstream error | timeout | denied | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...;`  
**Default**: `grpc_next_upstream error timeout;`  
**Context**: `http , server , location`  

Specifies in which cases a request should be passed to the next server:

### `grpc_next_upstream_timeout`

**Syntax**: `grpc_next_upstream_timeout time ;`  
**Default**: `grpc_next_upstream_timeout 0;`  
**Context**: `http , server , location`  

Limits the time during which a request can be passed to the next server . The 0 value turns off this limitation.

### `grpc_next_upstream_tries`

**Syntax**: `grpc_next_upstream_tries number ;`  
**Default**: `grpc_next_upstream_tries 0;`  
**Context**: `http , server , location`  

Limits the number of possible tries for passing a request to the next server . The 0 value turns off this limitation.

### `grpc_pass`

**Syntax**: `grpc_pass address ;`  
**Default**: `—`  
**Context**: `location , if in location`  

Sets the gRPC server address. The address can be specified as a domain name or IP address, and a port: grpc_pass localhost:9000; or as a UNIX-domain socket path: grpc_pass unix:/tmp/grpc.socket; Alternatively, the “ grpc:// ” scheme can be used:

### `grpc_pass_header`

**Syntax**: `grpc_pass_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Permits passing otherwise disabled header fields from a gRPC server to a client.

### `grpc_read_timeout`

**Syntax**: `grpc_read_timeout time ;`  
**Default**: `grpc_read_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading a response from the gRPC server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the gRPC server does not transmit anything within this time, the connection is closed.

### `grpc_send_timeout`

**Syntax**: `grpc_send_timeout time ;`  
**Default**: `grpc_send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a request to the gRPC server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the gRPC server does not receive anything within this time, the connection is closed.

### `grpc_set_header`

**Syntax**: `grpc_set_header field value ;`  
**Default**: `grpc_set_header Content-Length $content_length;`  
**Context**: `http , server , location`  

Allows redefining or appending fields to the request header passed to the gRPC server. The value can contain text, variables, and their combinations. These directives are inherited from the previous configuration level if and only if there are no grpc_set_header directives defined on the current level. If the value of a header field is an empty string then this field will not be passed to a gRPC server: grpc_set_header Accept-Encoding "";

### `grpc_ssl_certificate`

**Syntax**: `grpc_ssl_certificate file ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies a file with the certificate in the PEM format used for authentication to a gRPC SSL server.

### `grpc_ssl_certificate_key`

**Syntax**: `grpc_ssl_certificate_key file ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies a file with the secret key in the PEM format used for authentication to a gRPC SSL server. The value engine : name : id can be specified instead of the file , which loads a secret key with a specified id from the OpenSSL engine name . The value store : scheme : id can be specified instead of the file (1.29.0), which is used to load a secret key with a specified id and OpenSSL provider registered URI scheme , such as pkcs11 .

### `grpc_ssl_ciphers`

**Syntax**: `grpc_ssl_ciphers ciphers ;`  
**Default**: `grpc_ssl_ciphers DEFAULT;`  
**Context**: `http , server , location`  

Specifies the enabled ciphers for requests to a gRPC SSL server. The ciphers are specified in the format understood by the OpenSSL library. The full list can be viewed using the “ openssl ciphers ” command.

### `grpc_ssl_crl`

**Syntax**: `grpc_ssl_crl file ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies a file with revoked certificates (CRL) in the PEM format used to verify the certificate of the gRPC SSL server. When using intermediate certificates, their CRLs should be specified in the same file.

### `grpc_ssl_name`

**Syntax**: `grpc_ssl_name name ;`  
**Default**: `grpc_ssl_name host from grpc_pass;`  
**Context**: `http , server , location`  

Allows overriding the server name used to verify the certificate of the gRPC SSL server and to be passed through SNI when establishing a connection with the gRPC SSL server. By default, the host part from grpc_pass is used.

### `grpc_ssl_password_file`

**Syntax**: `grpc_ssl_password_file file ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies a file with passphrases for secret keys where each passphrase is specified on a separate line. Passphrases are tried in turn when loading the key.

### `grpc_ssl_protocols`

**Syntax**: `grpc_ssl_protocols [ SSLv2 ] [ SSLv3 ] [ TLSv1 ] [ TLSv1.1 ] [ TLSv1.2 ] [ TLSv1.3 ];`  
**Default**: `grpc_ssl_protocols TLSv1.2 TLSv1.3;`  
**Context**: `http , server , location`  

Enables the specified protocols for requests to a gRPC SSL server.

### `grpc_ssl_server_name`

**Syntax**: `grpc_ssl_server_name on | off ;`  
**Default**: `grpc_ssl_server_name off;`  
**Context**: `http , server , location`  

Enables or disables passing of the server name through TLS Server Name Indication extension (SNI, RFC 6066) when establishing a connection with the gRPC SSL server.

### `grpc_ssl_session_reuse`

**Syntax**: `grpc_ssl_session_reuse on | off ;`  
**Default**: `grpc_ssl_session_reuse on;`  
**Context**: `http , server , location`  

Determines whether SSL sessions can be reused when working with the gRPC server. If the errors “ digest check failed ” appear in the logs, try disabling session reuse.

### `grpc_ssl_trusted_certificate`

**Syntax**: `grpc_ssl_trusted_certificate file ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies a file with trusted CA certificates in the PEM format used to verify the certificate of the gRPC SSL server.

### `grpc_ssl_verify`

**Syntax**: `grpc_ssl_verify on | off ;`  
**Default**: `grpc_ssl_verify off;`  
**Context**: `http , server , location`  

Enables or disables verification of the gRPC SSL server certificate.

### `grpc_ssl_verify_depth`

**Syntax**: `grpc_ssl_verify_depth number ;`  
**Default**: `grpc_ssl_verify_depth 1;`  
**Context**: `http , server , location`  

Sets the verification depth in the gRPC SSL server certificates chain.
