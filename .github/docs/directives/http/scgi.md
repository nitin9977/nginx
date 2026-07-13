# Module ngx_http_scgi_module

**Source**: https://nginx.org/en/docs/http/ngx_http_scgi_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives scgi_allow_upstream scgi_bind scgi_bind_dynamic scgi_buffer_size scgi_buffering scgi_buffers scgi_busy_buffers_size scgi_cache scgi_cache_background_update scgi_cache_bypass scgi_cache_key scgi_cache_lock scgi_cache_lock_age scgi_cache_lock_timeout scgi_cache_max_range_offset scgi_cache_methods scgi_cache_min_uses scgi_cache_path scgi_cache_purge scgi_cache_revalidate scgi_cache_use_stale scgi_cache_valid scgi_connect_timeout scgi_force_ranges scgi_hide_header scgi_ignore_client_abort scgi_ignore_headers scgi_intercept_errors scgi_limit_rate scgi_max_temp_file_size scgi_next_upstream scgi_next_upstream_timeout scgi_next_upstream_tries scgi_no_cache scgi_param scgi_pass scgi_pass_header scgi_pass_request_body scgi_pass_request_headers scgi_read_timeout scgi_

## Example Configuration

```nginx
location / {
    include   scgi_params;
    scgi_pass localhost:9000;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `scgi_bind` | scgi_bind address [ transparent ] \| off ; | — | http , server , location |
| `scgi_buffer_size` | scgi_buffer_size size ; | scgi_buffer_size 4k\|8k; | http , server , location |
| `scgi_buffering` | scgi_buffering on \| off ; | scgi_buffering on; | http , server , location |
| `scgi_buffers` | scgi_buffers number size ; | scgi_buffers 8 4k\|8k; | http , server , location |
| `scgi_busy_buffers_size` | scgi_busy_buffers_size size ; | scgi_busy_buffers_size 8k\|16k; | http , server , location |
| `scgi_cache` | scgi_cache zone \| off ; | scgi_cache off; | http , server , location |
| `scgi_cache_bypass` | scgi_cache_bypass string ...; | — | http , server , location |
| `scgi_cache_key` | scgi_cache_key string ; | — | http , server , location |
| `scgi_cache_methods` | scgi_cache_methods GET \| HEAD \| POST ...; | scgi_cache_methods GET HEAD; | http , server , location |
| `scgi_cache_min_uses` | scgi_cache_min_uses number ; | scgi_cache_min_uses 1; | http , server , location |
| `scgi_cache_path` | scgi_cache_path path [ levels = levels ] [ use_temp_path = on \| off ] keys_zone = name : size [ inactive = time ] [ max | — | http |
| `scgi_cache_use_stale` | scgi_cache_use_stale error \| timeout \| invalid_header \| updating \| http_500 \| http_503 \| http_403 \| http_404 \| h | scgi_cache_use_stale off; | http , server , location |
| `scgi_cache_valid` | scgi_cache_valid [ code ...] time ; | — | http , server , location |
| `scgi_connect_timeout` | scgi_connect_timeout time ; | scgi_connect_timeout 60s; | http , server , location |
| `scgi_hide_header` | scgi_hide_header field ; | — | http , server , location |
| `scgi_ignore_client_abort` | scgi_ignore_client_abort on \| off ; | scgi_ignore_client_abort off; | http , server , location |
| `scgi_ignore_headers` | scgi_ignore_headers field ...; | — | http , server , location |
| `scgi_intercept_errors` | scgi_intercept_errors on \| off ; | scgi_intercept_errors off; | http , server , location |
| `scgi_max_temp_file_size` | scgi_max_temp_file_size size ; | scgi_max_temp_file_size 1024m; | http , server , location |
| `scgi_next_upstream` | scgi_next_upstream error \| timeout \| denied \| invalid_header \| http_500 \| http_503 \| http_403 \| http_404 \| http_ | scgi_next_upstream error timeout; | http , server , location |
| `scgi_no_cache` | scgi_no_cache string ...; | — | http , server , location |
| `scgi_param` | scgi_param parameter value [ if_not_empty ]; | scgi_param HTTP_HOST $host$is_request_port$request_port; | http , server , location |
| `scgi_pass` | scgi_pass address ; | — | location , if in location |
| `scgi_pass_header` | scgi_pass_header field ; | — | http , server , location |
| `scgi_pass_request_body` | scgi_pass_request_body on \| off ; | scgi_pass_request_body on; | http , server , location |
| `scgi_pass_request_headers` | scgi_pass_request_headers on \| off ; | scgi_pass_request_headers on; | http , server , location |
| `scgi_read_timeout` | scgi_read_timeout time ; | scgi_read_timeout 60s; | http , server , location |
| `scgi_send_timeout` | scgi_send_timeout time ; | scgi_send_timeout 60s; | http , server , location |
| `scgi_store` | scgi_store on \| off \| string ; | scgi_store off; | http , server , location |
| `scgi_store_access` | scgi_store_access users : permissions ...; | scgi_store_access user:rw; | http , server , location |
| `scgi_temp_file_write_size` | scgi_temp_file_write_size size ; | scgi_temp_file_write_size 8k\|16k; | http , server , location |
| `scgi_temp_path` | scgi_temp_path path [ level1 [ level2 [ level3 ]]]; | scgi_temp_path scgi_temp; | http , server , location |

## Directive Details

### `scgi_bind`

**Syntax**: `scgi_bind address [ transparent ] | off ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Makes outgoing connections to an SCGI server originate from the specified local IP address with an optional port (1.11.2). Parameter value can contain variables (1.3.12). The special value off (1.3.12) cancels the effect of the scgi_bind directive inherited from the previous configuration level, which allows the system to auto-assign the local IP address and port.

### `scgi_buffer_size`

**Syntax**: `scgi_buffer_size size ;`  
**Default**: `scgi_buffer_size 4k|8k;`  
**Context**: `http , server , location`  

Sets the size of the buffer used for reading the first part of the response received from the SCGI server. This part usually contains a small response header; if it exceeds the buffer size, the response is considered invalid . By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform. It can be made smaller, however.

### `scgi_buffering`

**Syntax**: `scgi_buffering on | off ;`  
**Default**: `scgi_buffering on;`  
**Context**: `http , server , location`  

Enables or disables buffering of responses from the SCGI server. When buffering is enabled, nginx receives a response from the SCGI server as soon as possible, saving it into the buffers set by the scgi_buffer_size and scgi_buffers directives. If the whole response does not fit into memory, a part of it can be saved to a temporary file on the disk. Writing to temporary files is controlled by the scgi_max_temp_file_size and scgi_temp_file_write_size directives. When buffering is disabled, the response is passed to a client synchronously, immediately as it is received. nginx will not try to read the whole response from the SCGI server. The maximum size of the data that nginx can receive from the server at a time is set by the scgi_buffer_size directive.

### `scgi_buffers`

**Syntax**: `scgi_buffers number size ;`  
**Default**: `scgi_buffers 8 4k|8k;`  
**Context**: `http , server , location`  

Sets the number and size of the buffers used for reading a response from the SCGI server, for a single connection. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.

### `scgi_busy_buffers_size`

**Syntax**: `scgi_busy_buffers_size size ;`  
**Default**: `scgi_busy_buffers_size 8k|16k;`  
**Context**: `http , server , location`  

When buffering of responses from the SCGI server is enabled, limits the total size of buffers that can be busy sending a response to the client while the response is not yet fully read. In the meantime, the rest of the buffers can be used for reading the response and, if needed, buffering part of the response to a temporary file. By default, size is limited by the size of two buffers set by the scgi_buffer_size and scgi_buffers directives.

### `scgi_cache`

**Syntax**: `scgi_cache zone | off ;`  
**Default**: `scgi_cache off;`  
**Context**: `http , server , location`  

Defines a shared memory zone used for caching. The same zone can be used in several places. Parameter value can contain variables (1.7.9). The off parameter disables caching inherited from the previous configuration level.

### `scgi_cache_bypass`

**Syntax**: `scgi_cache_bypass string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be taken from a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be taken from the cache: scgi_cache_bypass $cookie_nocache $arg_nocache$arg_comment; scgi_cache_bypass $http_pragma $http_authorization; Can be used along with the scgi_no_cache directive.

### `scgi_cache_key`

**Syntax**: `scgi_cache_key string ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines a key for caching, for example scgi_cache_key localhost:9000$request_uri;

### `scgi_cache_methods`

**Syntax**: `scgi_cache_methods GET | HEAD | POST ...;`  
**Default**: `scgi_cache_methods GET HEAD;`  
**Context**: `http , server , location`  

If the client request method is listed in this directive then the response will be cached. “ GET ” and “ HEAD ” methods are always added to the list, though it is recommended to specify them explicitly. See also the scgi_no_cache directive.

### `scgi_cache_min_uses`

**Syntax**: `scgi_cache_min_uses number ;`  
**Default**: `scgi_cache_min_uses 1;`  
**Context**: `http , server , location`  

Sets the number of requests after which the response will be cached.

### `scgi_cache_path`

**Syntax**: `scgi_cache_path path [ levels = levels ] [ use_temp_path = on | off ] keys_zone = name : size [ inactive = time ] [ max_size = size ] [ min_free = size ] [ manager_files = number ] [ manager_sleep = time ] [ manager_threshold = time ] [ loader_files = number ] [ loader_sleep = time ] [ loader_threshold = time ] [ purger = on | off ] [ purger_files = number ] [ purger_sleep = time ] [ purger_threshold = time ];`  
**Default**: `—`  
**Context**: `http`  

Sets the path and other parameters of a cache. Cache data are stored in files. The file name in a cache is a result of applying the MD5 function to the cache key . The levels parameter defines hierarchy levels of a cache: from 1 to 3, each level accepts values 1 or 2. For example, in the following configuration scgi_cache_path /data/nginx/cache levels=1:2 keys_zone=one:10m; file names in a cache will look like this: /data/nginx/cache/ c / 29 /b7f54b2df7773722d382f4809d650 29c

### `scgi_cache_use_stale`

**Syntax**: `scgi_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_503 | http_403 | http_404 | http_429 | off ...;`  
**Default**: `scgi_cache_use_stale off;`  
**Context**: `http , server , location`  

Determines in which cases a stale cached response can be used when an error occurs during communication with the SCGI server. The directive’s parameters match the parameters of the scgi_next_upstream directive. The error parameter also permits using a stale cached response if an SCGI server to process a request cannot be selected.

### `scgi_cache_valid`

**Syntax**: `scgi_cache_valid [ code ...] time ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets caching time for different response codes. For example, the following directives scgi_cache_valid 200 302 10m; scgi_cache_valid 404 1m; set 10 minutes of caching for responses with codes 200 and 302 and 1 minute for responses with code 404. If only caching time is specified

### `scgi_connect_timeout`

**Syntax**: `scgi_connect_timeout time ;`  
**Default**: `scgi_connect_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for establishing a connection with an SCGI server. It should be noted that this timeout cannot usually exceed 75 seconds.

### `scgi_hide_header`

**Syntax**: `scgi_hide_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

By default, nginx does not pass the header fields “Status” and “X-Accel-...” from the response of an SCGI server to a client. The scgi_hide_header directive sets additional fields that will not be passed. If, on the contrary, the passing of fields needs to be permitted, the scgi_pass_header directive can be used.

### `scgi_ignore_client_abort`

**Syntax**: `scgi_ignore_client_abort on | off ;`  
**Default**: `scgi_ignore_client_abort off;`  
**Context**: `http , server , location`  

Determines whether the connection with an SCGI server should be closed when a client closes the connection without waiting for a response.

### `scgi_ignore_headers`

**Syntax**: `scgi_ignore_headers field ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Disables processing of certain response header fields from the SCGI server. The following fields can be ignored: “X-Accel-Redirect”, “X-Accel-Expires”, “X-Accel-Limit-Rate” (1.1.6), “X-Accel-Buffering” (1.1.6), “X-Accel-Charset” (1.1.6), “Expires”, “Cache-Control”, “Set-Cookie” (0.8.44), and “Vary” (1.7.7). If not disabled, processing of these header fields has the following effect:

### `scgi_intercept_errors`

**Syntax**: `scgi_intercept_errors on | off ;`  
**Default**: `scgi_intercept_errors off;`  
**Context**: `http , server , location`  

Determines whether an SCGI server responses with codes greater than or equal to 300 should be passed to a client or be intercepted and redirected to nginx for processing with the error_page directive.

### `scgi_max_temp_file_size`

**Syntax**: `scgi_max_temp_file_size size ;`  
**Default**: `scgi_max_temp_file_size 1024m;`  
**Context**: `http , server , location`  

When buffering of responses from the SCGI server is enabled, and the whole response does not fit into the buffers set by the scgi_buffer_size and scgi_buffers directives, a part of the response can be saved to a temporary file. This directive sets the maximum size of the temporary file. The size of data written to the temporary file at a time is set by the scgi_temp_file_write_size directive. The zero value disables buffering of responses to temporary files.

### `scgi_next_upstream`

**Syntax**: `scgi_next_upstream error | timeout | denied | invalid_header | http_500 | http_503 | http_403 | http_404 | http_429 | non_idempotent | off ...;`  
**Default**: `scgi_next_upstream error timeout;`  
**Context**: `http , server , location`  

Specifies in which cases a request should be passed to the next server:

### `scgi_no_cache`

**Syntax**: `scgi_no_cache string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be saved to a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be saved: scgi_no_cache $cookie_nocache $arg_nocache$arg_comment; scgi_no_cache $http_pragma $http_authorization; Can be used along with the scgi_cache_bypass directive.

### `scgi_param`

**Syntax**: `scgi_param parameter value [ if_not_empty ];`  
**Default**: `scgi_param HTTP_HOST $host$is_request_port$request_port;`  
**Context**: `http , server , location`  

Sets a parameter that should be passed to the SCGI server. The value can contain text, variables, and their combination. These directives are inherited from the previous configuration level if and only if there are no scgi_param directives defined on the current level. Standard CGI environment variables should be provided as SCGI headers, see the scgi_params file provided in the distribution: location / { include scgi_params; ... }

### `scgi_pass`

**Syntax**: `scgi_pass address ;`  
**Default**: `—`  
**Context**: `location , if in location`  

Sets the address of an SCGI server. The address can be specified as a domain name or IP address, and a port: scgi_pass localhost:9000; or as a UNIX-domain socket path: scgi_pass unix:/tmp/scgi.socket;

### `scgi_pass_header`

**Syntax**: `scgi_pass_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Permits passing otherwise disabled header fields from an SCGI server to a client.

### `scgi_pass_request_body`

**Syntax**: `scgi_pass_request_body on | off ;`  
**Default**: `scgi_pass_request_body on;`  
**Context**: `http , server , location`  

Indicates whether the original request body is passed to the SCGI server. See also the scgi_pass_request_headers directive.

### `scgi_pass_request_headers`

**Syntax**: `scgi_pass_request_headers on | off ;`  
**Default**: `scgi_pass_request_headers on;`  
**Context**: `http , server , location`  

Indicates whether the header fields of the original request are passed to the SCGI server. See also the scgi_pass_request_body directive.

### `scgi_read_timeout`

**Syntax**: `scgi_read_timeout time ;`  
**Default**: `scgi_read_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading a response from the SCGI server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the SCGI server does not transmit anything within this time, the connection is closed.

### `scgi_send_timeout`

**Syntax**: `scgi_send_timeout time ;`  
**Default**: `scgi_send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a request to the SCGI server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the SCGI server does not receive anything within this time, the connection is closed.

### `scgi_store`

**Syntax**: `scgi_store on | off | string ;`  
**Default**: `scgi_store off;`  
**Context**: `http , server , location`  

Enables saving of files to a disk. The on parameter saves files with paths corresponding to the directives alias or root . The off parameter disables saving of files. In addition, the file name can be set explicitly using the string with variables: scgi_store /data/www$original_uri; The modification time of files is set according to the received “Last-Modified” response header field. The response is first written to a temporary file, and then the file is renamed. Starting from version 0.8.9, temporary files and the persistent store can be put on different file systems. However, be aware that in this case a file is copied across two file systems instead of the cheap renaming operation. It is thus recommended that for any given location both saved files and a directory holding temporary files, set by the scgi_temp_path directive, are put on the same file system.

### `scgi_store_access`

**Syntax**: `scgi_store_access users : permissions ...;`  
**Default**: `scgi_store_access user:rw;`  
**Context**: `http , server , location`  

Sets access permissions for newly created files and directories, e.g.: scgi_store_access user:rw group:rw all:r; If any group or all access permissions are specified then user permissions may be omitted:

### `scgi_temp_file_write_size`

**Syntax**: `scgi_temp_file_write_size size ;`  
**Default**: `scgi_temp_file_write_size 8k|16k;`  
**Context**: `http , server , location`  

Limits the size of data written to a temporary file at a time, when buffering of responses from the SCGI server to temporary files is enabled. By default, size is limited by two buffers set by the scgi_buffer_size and scgi_buffers directives. The maximum size of a temporary file is set by the scgi_max_temp_file_size directive.

### `scgi_temp_path`

**Syntax**: `scgi_temp_path path [ level1 [ level2 [ level3 ]]];`  
**Default**: `scgi_temp_path scgi_temp;`  
**Context**: `http , server , location`  

Defines a directory for storing temporary files with data received from SCGI servers. Up to three-level subdirectory hierarchy can be used underneath the specified directory. For example, in the following configuration scgi_temp_path /spool/nginx/scgi_temp 1 2; a temporary file might look like this: /spool/nginx/scgi_temp/ 7 / 45 /00000123 457
