# Module ngx_http_proxy_module

**Source**: https://nginx.org/en/docs/http/ngx_http_proxy_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives proxy_allow_upstream proxy_bind proxy_bind_dynamic proxy_buffer_size proxy_buffering proxy_buffers proxy_busy_buffers_size proxy_cache proxy_cache_background_update proxy_cache_bypass proxy_cache_convert_head proxy_cache_key proxy_cache_lock proxy_cache_lock_age proxy_cache_lock_timeout proxy_cache_max_range_offset proxy_cache_methods proxy_cache_min_uses proxy_cache_path proxy_cache_purge proxy_cache_revalidate proxy_cache_use_stale proxy_cache_valid proxy_connect_timeout proxy_cookie_domain proxy_cookie_flags proxy_cookie_path proxy_force_ranges proxy_headers_hash_bucket_size proxy_headers_hash_max_size proxy_hide_header proxy_http_version proxy_ignore_client_abort proxy_ignore_headers proxy_intercept_errors proxy_limit_rate proxy_max_temp_file_size proxy

## Example Configuration

```nginx
location / {
    proxy_pass       http://localhost:8000;
    proxy_set_header Host      $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `proxy_buffer_size` | proxy_buffer_size size ; | proxy_buffer_size 4k\|8k; | http , server , location |
| `proxy_buffering` | proxy_buffering on \| off ; | proxy_buffering on; | http , server , location |
| `proxy_buffers` | proxy_buffers number size ; | proxy_buffers 8 4k\|8k; | http , server , location |
| `proxy_busy_buffers_size` | proxy_busy_buffers_size size ; | proxy_busy_buffers_size 8k\|16k; | http , server , location |
| `proxy_cache` | proxy_cache zone \| off ; | proxy_cache off; | http , server , location |
| `proxy_cache_bypass` | proxy_cache_bypass string ...; | — | http , server , location |
| `proxy_cache_key` | proxy_cache_key string ; | proxy_cache_key $scheme$proxy_host$request_uri; | http , server , location |
| `proxy_cache_min_uses` | proxy_cache_min_uses number ; | proxy_cache_min_uses 1; | http , server , location |
| `proxy_cache_path` | proxy_cache_path path [ levels = levels ] [ use_temp_path = on \| off ] keys_zone = name : size [ inactive = time ] [ ma | — | http |
| `proxy_cache_use_stale` | proxy_cache_use_stale error \| timeout \| invalid_header \| updating \| http_500 \| http_502 \| http_503 \| http_504 \|  | proxy_cache_use_stale off; | http , server , location |
| `proxy_cache_valid` | proxy_cache_valid [ code ...] time ; | — | http , server , location |
| `proxy_connect_timeout` | proxy_connect_timeout time ; | proxy_connect_timeout 60s; | http , server , location |
| `proxy_headers_hash_bucket_size` | proxy_headers_hash_bucket_size size ; | proxy_headers_hash_bucket_size 64; | http , server , location |
| `proxy_headers_hash_max_size` | proxy_headers_hash_max_size size ; | proxy_headers_hash_max_size 512; | http , server , location |
| `proxy_hide_header` | proxy_hide_header field ; | — | http , server , location |
| `proxy_ignore_client_abort` | proxy_ignore_client_abort on \| off ; | proxy_ignore_client_abort off; | http , server , location |
| `proxy_ignore_headers` | proxy_ignore_headers field ...; | — | http , server , location |
| `proxy_intercept_errors` | proxy_intercept_errors on \| off ; | proxy_intercept_errors off; | http , server , location |
| `proxy_max_temp_file_size` | proxy_max_temp_file_size size ; | proxy_max_temp_file_size 1024m; | http , server , location |
| `proxy_method` | proxy_method method ; | — | http , server , location |
| `proxy_next_upstream` | proxy_next_upstream error \| timeout \| denied \| invalid_header \| http_500 \| http_502 \| http_503 \| http_504 \| http | proxy_next_upstream error timeout; | http , server , location |
| `proxy_no_cache` | proxy_no_cache string ...; | — | http , server , location |
| `proxy_pass` | proxy_pass URL ; | — | location , if in location , limit_except |
| `proxy_pass_header` | proxy_pass_header field ; | — | http , server , location |
| `proxy_pass_request_body` | proxy_pass_request_body on \| off ; | proxy_pass_request_body on; | http , server , location |
| `proxy_pass_request_headers` | proxy_pass_request_headers on \| off ; | proxy_pass_request_headers on; | http , server , location |
| `proxy_read_timeout` | proxy_read_timeout time ; | proxy_read_timeout 60s; | http , server , location |
| `proxy_redirect` | proxy_redirect default ; proxy_redirect off ; proxy_redirect redirect replacement ; | proxy_redirect default; | http , server , location |
| `proxy_send_lowat` | proxy_send_lowat size ; | proxy_send_lowat 0; | http , server , location |
| `proxy_send_timeout` | proxy_send_timeout time ; | proxy_send_timeout 60s; | http , server , location |
| `proxy_set_body` | proxy_set_body value ; | — | http , server , location |
| `proxy_set_header` | proxy_set_header field value ; | proxy_set_header Host $proxy_host; proxy_set_header Connecti | http , server , location |
| `proxy_ssl_session_reuse` | proxy_ssl_session_reuse on \| off ; | proxy_ssl_session_reuse on; | http , server , location |
| `proxy_store` | proxy_store on \| off \| string ; | proxy_store off; | http , server , location |
| `proxy_store_access` | proxy_store_access users : permissions ...; | proxy_store_access user:rw; | http , server , location |
| `proxy_temp_file_write_size` | proxy_temp_file_write_size size ; | proxy_temp_file_write_size 8k\|16k; | http , server , location |
| `proxy_temp_path` | proxy_temp_path path [ level1 [ level2 [ level3 ]]]; | proxy_temp_path proxy_temp; | http , server , location |

## Directive Details

### `proxy_buffer_size`

**Syntax**: `proxy_buffer_size size ;`  
**Default**: `proxy_buffer_size 4k|8k;`  
**Context**: `http , server , location`  

Sets the size of the buffer used for reading the first part of the response received from the proxied server. This part usually contains a small response header; if it exceeds the buffer size, the response is considered invalid . By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform. It can be made smaller, however.

### `proxy_buffering`

**Syntax**: `proxy_buffering on | off ;`  
**Default**: `proxy_buffering on;`  
**Context**: `http , server , location`  

Enables or disables buffering of responses from the proxied server. When buffering is enabled, nginx receives a response from the proxied server as soon as possible, saving it into the buffers set by the proxy_buffer_size and proxy_buffers directives. If the whole response does not fit into memory, a part of it can be saved to a temporary file on the disk. Writing to temporary files is controlled by the proxy_max_temp_file_size and proxy_temp_file_write_size directives. When buffering is disabled, the response is passed to a client synchronously, immediately as it is received. nginx will not try to read the whole response from the proxied server. The maximum size of the data that nginx can receive from the server at a time is set by the proxy_buffer_size directive.

### `proxy_buffers`

**Syntax**: `proxy_buffers number size ;`  
**Default**: `proxy_buffers 8 4k|8k;`  
**Context**: `http , server , location`  

Sets the number and size of the buffers used for reading a response from the proxied server, for a single connection. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.

### `proxy_busy_buffers_size`

**Syntax**: `proxy_busy_buffers_size size ;`  
**Default**: `proxy_busy_buffers_size 8k|16k;`  
**Context**: `http , server , location`  

When buffering of responses from the proxied server is enabled, limits the total size of buffers that can be busy sending a response to the client while the response is not yet fully read. In the meantime, the rest of the buffers can be used for reading the response and, if needed, buffering part of the response to a temporary file. By default, size is limited by the size of two buffers set by the proxy_buffer_size and proxy_buffers directives.

### `proxy_cache`

**Syntax**: `proxy_cache zone | off ;`  
**Default**: `proxy_cache off;`  
**Context**: `http , server , location`  

Defines a shared memory zone used for caching. The same zone can be used in several places. Parameter value can contain variables (1.7.9). The off parameter disables caching inherited from the previous configuration level.

### `proxy_cache_bypass`

**Syntax**: `proxy_cache_bypass string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be taken from a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be taken from the cache: proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment; proxy_cache_bypass $http_pragma $http_authorization; Can be used along with the proxy_no_cache directive.

### `proxy_cache_key`

**Syntax**: `proxy_cache_key string ;`  
**Default**: `proxy_cache_key $scheme$proxy_host$request_uri;`  
**Context**: `http , server , location`  

Defines a key for caching, for example proxy_cache_key "$host$request_uri $cookie_user"; By default, the directive’s value is close to the string proxy_cache_key $scheme$proxy_host$uri$is_args$args;

### `proxy_cache_min_uses`

**Syntax**: `proxy_cache_min_uses number ;`  
**Default**: `proxy_cache_min_uses 1;`  
**Context**: `http , server , location`  

Sets the number of requests after which the response will be cached.

### `proxy_cache_path`

**Syntax**: `proxy_cache_path path [ levels = levels ] [ use_temp_path = on | off ] keys_zone = name : size [ inactive = time ] [ max_size = size ] [ min_free = size ] [ manager_files = number ] [ manager_sleep = time ] [ manager_threshold = time ] [ loader_files = number ] [ loader_sleep = time ] [ loader_threshold = time ] [ purger = on | off ] [ purger_files = number ] [ purger_sleep = time ] [ purger_threshold = time ];`  
**Default**: `—`  
**Context**: `http`  

Sets the path and other parameters of a cache. Cache data are stored in files. The file name in a cache is a result of applying the MD5 function to the cache key . The levels parameter defines hierarchy levels of a cache: from 1 to 3, each level accepts values 1 or 2. For example, in the following configuration proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=one:10m; file names in a cache will look like this: /data/nginx/cache/ c / 29 /b7f54b2df7773722d382f4809d650 29c

### `proxy_cache_use_stale`

**Syntax**: `proxy_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | off ...;`  
**Default**: `proxy_cache_use_stale off;`  
**Context**: `http , server , location`  

Determines in which cases a stale cached response can be used during communication with the proxied server. The directive’s parameters match the parameters of the proxy_next_upstream directive. The error parameter also permits using a stale cached response if a proxied server to process a request cannot be selected.

### `proxy_cache_valid`

**Syntax**: `proxy_cache_valid [ code ...] time ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets caching time for different response codes. For example, the following directives proxy_cache_valid 200 302 10m; proxy_cache_valid 404 1m; set 10 minutes of caching for responses with codes 200 and 302 and 1 minute for responses with code 404. If only caching time is specified

### `proxy_connect_timeout`

**Syntax**: `proxy_connect_timeout time ;`  
**Default**: `proxy_connect_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for establishing a connection with a proxied server. It should be noted that this timeout cannot usually exceed 75 seconds.

### `proxy_headers_hash_bucket_size`

**Syntax**: `proxy_headers_hash_bucket_size size ;`  
**Default**: `proxy_headers_hash_bucket_size 64;`  
**Context**: `http , server , location`  

Sets the bucket size for hash tables used by the proxy_hide_header and proxy_set_header directives. The details of setting up hash tables are provided in a separate document .

### `proxy_headers_hash_max_size`

**Syntax**: `proxy_headers_hash_max_size size ;`  
**Default**: `proxy_headers_hash_max_size 512;`  
**Context**: `http , server , location`  

Sets the maximum size of hash tables used by the proxy_hide_header and proxy_set_header directives. The details of setting up hash tables are provided in a separate document .

### `proxy_hide_header`

**Syntax**: `proxy_hide_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

By default, nginx does not pass the header fields “Date”, “Server”, “X-Pad”, and “X-Accel-...” from the response of a proxied server to a client. The proxy_hide_header directive sets additional fields that will not be passed. If, on the contrary, the passing of fields needs to be permitted, the proxy_pass_header directive can be used.

### `proxy_ignore_client_abort`

**Syntax**: `proxy_ignore_client_abort on | off ;`  
**Default**: `proxy_ignore_client_abort off;`  
**Context**: `http , server , location`  

Determines whether the connection with a proxied server should be closed when a client closes the connection without waiting for a response.

### `proxy_ignore_headers`

**Syntax**: `proxy_ignore_headers field ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Disables processing of certain response header fields from the proxied server. The following fields can be ignored: “X-Accel-Redirect”, “X-Accel-Expires”, “X-Accel-Limit-Rate” (1.1.6), “X-Accel-Buffering” (1.1.6), “X-Accel-Charset” (1.1.6), “Expires”, “Cache-Control”, “Set-Cookie” (0.8.44), and “Vary” (1.7.7). If not disabled, processing of these header fields has the following effect:

### `proxy_intercept_errors`

**Syntax**: `proxy_intercept_errors on | off ;`  
**Default**: `proxy_intercept_errors off;`  
**Context**: `http , server , location`  

Determines whether proxied responses with codes greater than or equal to 300 should be passed to a client or be intercepted and redirected to nginx for processing with the error_page directive.

### `proxy_max_temp_file_size`

**Syntax**: `proxy_max_temp_file_size size ;`  
**Default**: `proxy_max_temp_file_size 1024m;`  
**Context**: `http , server , location`  

When buffering of responses from the proxied server is enabled, and the whole response does not fit into the buffers set by the proxy_buffer_size and proxy_buffers directives, a part of the response can be saved to a temporary file. This directive sets the maximum size of the temporary file. The size of data written to the temporary file at a time is set by the proxy_temp_file_write_size directive. The zero value disables buffering of responses to temporary files.

### `proxy_method`

**Syntax**: `proxy_method method ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies the HTTP method to use in requests forwarded to the proxied server instead of the method from the client request. Parameter value can contain variables (1.11.6).

### `proxy_next_upstream`

**Syntax**: `proxy_next_upstream error | timeout | denied | invalid_header | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | non_idempotent | off ...;`  
**Default**: `proxy_next_upstream error timeout;`  
**Context**: `http , server , location`  

Specifies in which cases a request should be passed to the next server:

### `proxy_no_cache`

**Syntax**: `proxy_no_cache string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be saved to a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be saved: proxy_no_cache $cookie_nocache $arg_nocache$arg_comment; proxy_no_cache $http_pragma $http_authorization; Can be used along with the proxy_cache_bypass directive.

### `proxy_pass`

**Syntax**: `proxy_pass URL ;`  
**Default**: `—`  
**Context**: `location , if in location , limit_except`  

Sets the protocol and address of a proxied server and an optional URI to which a location should be mapped. As a protocol, “ http ” or “ https ” can be specified. The address can be specified as a domain name or IP address, and an optional port: proxy_pass http://localhost:8000/uri/; or as a UNIX-domain socket path specified after the word “ unix ” and enclosed in colons: proxy_pass http://unix:/tmp/backend.socket:/uri/;

### `proxy_pass_header`

**Syntax**: `proxy_pass_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Permits passing otherwise disabled header fields from a proxied server to a client.

### `proxy_pass_request_body`

**Syntax**: `proxy_pass_request_body on | off ;`  
**Default**: `proxy_pass_request_body on;`  
**Context**: `http , server , location`  

Indicates whether the original request body is passed to the proxied server. location /x-accel-redirect-here/ { proxy_method GET; proxy_pass_request_body off; proxy_set_header Content-Length ""; proxy_pass ... } See also the proxy_set_header and proxy_pass_request_headers directives.

### `proxy_pass_request_headers`

**Syntax**: `proxy_pass_request_headers on | off ;`  
**Default**: `proxy_pass_request_headers on;`  
**Context**: `http , server , location`  

Indicates whether the header fields of the original request are passed to the proxied server. location /x-accel-redirect-here/ { proxy_method GET; proxy_pass_request_headers off; proxy_pass_request_body off; proxy_pass ... } See also the proxy_set_header and proxy_pass_request_body directives.

### `proxy_read_timeout`

**Syntax**: `proxy_read_timeout time ;`  
**Default**: `proxy_read_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading a response from the proxied server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the proxied server does not transmit anything within this time, the connection is closed.

### `proxy_redirect`

**Syntax**: `proxy_redirect default ; proxy_redirect off ; proxy_redirect redirect replacement ;`  
**Default**: `proxy_redirect default;`  
**Context**: `http , server , location`  

Sets the text that should be changed in the “Location” and “Refresh” header fields of a proxied server response. Suppose a proxied server returned the header field “ Location: http://localhost:8000/two/some/uri/ ”. The directive proxy_redirect http://localhost:8000/two/ http://frontend/one/; will rewrite this string to “ Location: http://frontend/one/some/uri/ ”. A server name may be omitted in the replacement string:

### `proxy_send_lowat`

**Syntax**: `proxy_send_lowat size ;`  
**Default**: `proxy_send_lowat 0;`  
**Context**: `http , server , location`  

If the directive is set to a non-zero value, nginx will try to minimize the number of send operations on outgoing connections to a proxied server by using either NOTE_LOWAT flag of the kqueue method, or the SO_SNDLOWAT socket option, with the specified size . This directive is ignored on Linux, Solaris, and Windows.

### `proxy_send_timeout`

**Syntax**: `proxy_send_timeout time ;`  
**Default**: `proxy_send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a request to the proxied server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the proxied server does not receive anything within this time, the connection is closed.

### `proxy_set_body`

**Syntax**: `proxy_set_body value ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Allows redefining the request body passed to the proxied server. The value can contain text, variables, and their combination.

### `proxy_set_header`

**Syntax**: `proxy_set_header field value ;`  
**Default**: `proxy_set_header Host $proxy_host; proxy_set_header Connection close;`  
**Context**: `http , server , location`  

Allows redefining or appending fields to the request header passed to the proxied server. The value can contain text, variables, and their combinations. These directives are inherited from the previous configuration level if and only if there are no proxy_set_header directives defined on the current level. By default, the header fields “Host” and “Connection” from the original request are not passed to the proxied server. If HTTP/1.0 or HTTP/1.1 is enabled for proxying, these fields are redefined: proxy_set_header Host $proxy_host; proxy_set_header Connection close; For HTTP/2, the “:authority” pseudo-header field with the $proxy_host value is sent by default, unless it is replaced with an explicit “Host” header field.

### `proxy_ssl_session_reuse`

**Syntax**: `proxy_ssl_session_reuse on | off ;`  
**Default**: `proxy_ssl_session_reuse on;`  
**Context**: `http , server , location`  

Determines whether SSL sessions can be reused when working with the proxied server. If the errors “ digest check failed ” appear in the logs, try disabling session reuse.

### `proxy_store`

**Syntax**: `proxy_store on | off | string ;`  
**Default**: `proxy_store off;`  
**Context**: `http , server , location`  

Enables saving of files to a disk. The on parameter saves files with paths corresponding to the directives alias or root . The off parameter disables saving of files. In addition, the file name can be set explicitly using the string with variables: proxy_store /data/www$original_uri; The modification time of files is set according to the received “Last-Modified” response header field. The response is first written to a temporary file, and then the file is renamed. Starting from version 0.8.9, temporary files and the persistent store can be put on different file systems. However, be aware that in this case a file is copied across two file systems instead of the cheap renaming operation. It is thus recommended that for any given location both saved files and a directory holding temporary files, set by the proxy_temp_path directive, are put on the same file system.

### `proxy_store_access`

**Syntax**: `proxy_store_access users : permissions ...;`  
**Default**: `proxy_store_access user:rw;`  
**Context**: `http , server , location`  

Sets access permissions for newly created files and directories, e.g.: proxy_store_access user:rw group:rw all:r; If any group or all access permissions are specified then user permissions may be omitted:

### `proxy_temp_file_write_size`

**Syntax**: `proxy_temp_file_write_size size ;`  
**Default**: `proxy_temp_file_write_size 8k|16k;`  
**Context**: `http , server , location`  

Limits the size of data written to a temporary file at a time, when buffering of responses from the proxied server to temporary files is enabled. By default, size is limited by two buffers set by the proxy_buffer_size and proxy_buffers directives. The maximum size of a temporary file is set by the proxy_max_temp_file_size directive.

### `proxy_temp_path`

**Syntax**: `proxy_temp_path path [ level1 [ level2 [ level3 ]]];`  
**Default**: `proxy_temp_path proxy_temp;`  
**Context**: `http , server , location`  

Defines a directory for storing temporary files with data received from proxied servers. Up to three-level subdirectory hierarchy can be used underneath the specified directory. For example, in the following configuration proxy_temp_path /spool/nginx/proxy_temp 1 2; a temporary file might look like this: /spool/nginx/proxy_temp/ 7 / 45 /00000123 457
