# Module ngx_http_memcached_module

**Source**: https://nginx.org/en/docs/http/ngx_http_memcached_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives memcached_allow_upstream memcached_bind memcached_bind_dynamic memcached_buffer_size memcached_connect_timeout memcached_gzip_flag memcached_next_upstream memcached_next_upstream_timeout memcached_next_upstream_tries memcached_pass memcached_read_timeout memcached_send_timeout memcached_socket_keepalive Embedded Variables The ngx_http_memcached_module module is used to obtain responses from a memcached server. The key is set in the $memcached_key variable. A response should be put in memcached in advance by means external to nginx.

## Example Configuration

```nginx
server {
    location / {
        set            $memcached_key "$uri?$args";
        memcached_pass host:11211;
        error_page     404 502 504 = @fallback;
    }

    location @fallback {
        proxy_pass     http://backend;
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `memcached_buffer_size` | memcached_buffer_size size ; | memcached_buffer_size 4k\|8k; | http , server , location |
| `memcached_connect_timeout` | memcached_connect_timeout time ; | memcached_connect_timeout 60s; | http , server , location |
| `memcached_next_upstream` | memcached_next_upstream error \| timeout \| denied \| invalid_response \| not_found \| off ...; | memcached_next_upstream error timeout; | http , server , location |
| `memcached_pass` | memcached_pass address ; | — | location , if in location |
| `memcached_read_timeout` | memcached_read_timeout time ; | memcached_read_timeout 60s; | http , server , location |
| `memcached_send_timeout` | memcached_send_timeout time ; | memcached_send_timeout 60s; | http , server , location |

## Directive Details

### `memcached_buffer_size`

**Syntax**: `memcached_buffer_size size ;`  
**Default**: `memcached_buffer_size 4k|8k;`  
**Context**: `http , server , location`  

Sets the size of the buffer used for reading the response received from the memcached server. The response is passed to the client synchronously, as soon as it is received.

### `memcached_connect_timeout`

**Syntax**: `memcached_connect_timeout time ;`  
**Default**: `memcached_connect_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for establishing a connection with a memcached server. It should be noted that this timeout cannot usually exceed 75 seconds.

### `memcached_next_upstream`

**Syntax**: `memcached_next_upstream error | timeout | denied | invalid_response | not_found | off ...;`  
**Default**: `memcached_next_upstream error timeout;`  
**Context**: `http , server , location`  

Specifies in which cases a request should be passed to the next server:

### `memcached_pass`

**Syntax**: `memcached_pass address ;`  
**Default**: `—`  
**Context**: `location , if in location`  

Sets the memcached server address. The address can be specified as a domain name or IP address, and a port: memcached_pass localhost:11211; or as a UNIX-domain socket path: memcached_pass unix:/tmp/memcached.socket;

### `memcached_read_timeout`

**Syntax**: `memcached_read_timeout time ;`  
**Default**: `memcached_read_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading a response from the memcached server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the memcached server does not transmit anything within this time, the connection is closed.

### `memcached_send_timeout`

**Syntax**: `memcached_send_timeout time ;`  
**Default**: `memcached_send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a request to the memcached server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the memcached server does not receive anything within this time, the connection is closed.
