# Module ngx_http_uwsgi_module

**Source**: https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives uwsgi_allow_upstream uwsgi_bind uwsgi_bind_dynamic uwsgi_buffer_size uwsgi_buffering uwsgi_buffers uwsgi_busy_buffers_size uwsgi_cache uwsgi_cache_background_update uwsgi_cache_bypass uwsgi_cache_key uwsgi_cache_lock uwsgi_cache_lock_age uwsgi_cache_lock_timeout uwsgi_cache_max_range_offset uwsgi_cache_methods uwsgi_cache_min_uses uwsgi_cache_path uwsgi_cache_purge uwsgi_cache_revalidate uwsgi_cache_use_stale uwsgi_cache_valid uwsgi_connect_timeout uwsgi_force_ranges uwsgi_hide_header uwsgi_ignore_client_abort uwsgi_ignore_headers uwsgi_intercept_errors uwsgi_limit_rate uwsgi_max_temp_file_size uwsgi_modifier1 uwsgi_modifier2 uwsgi_next_upstream uwsgi_next_upstream_timeout uwsgi_next_upstream_tries uwsgi_no_cache uwsgi_param uwsgi_pass uwsgi_pass_header uws

## Example Configuration

```nginx
location / {
    include    uwsgi_params;
    uwsgi_pass localhost:9000;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `uwsgi_bind` | uwsgi_bind address [ transparent ] \| off ; | — | http , server , location |
| `uwsgi_buffer_size` | uwsgi_buffer_size size ; | uwsgi_buffer_size 4k\|8k; | http , server , location |
| `uwsgi_buffering` | uwsgi_buffering on \| off ; | uwsgi_buffering on; | http , server , location |
| `uwsgi_buffers` | uwsgi_buffers number size ; | uwsgi_buffers 8 4k\|8k; | http , server , location |
| `uwsgi_busy_buffers_size` | uwsgi_busy_buffers_size size ; | uwsgi_busy_buffers_size 8k\|16k; | http , server , location |
| `uwsgi_cache` | uwsgi_cache zone \| off ; | uwsgi_cache off; | http , server , location |
| `uwsgi_cache_bypass` | uwsgi_cache_bypass string ...; | — | http , server , location |
| `uwsgi_cache_key` | uwsgi_cache_key string ; | — | http , server , location |
| `uwsgi_cache_methods` | uwsgi_cache_methods GET \| HEAD \| POST ...; | uwsgi_cache_methods GET HEAD; | http , server , location |
| `uwsgi_cache_min_uses` | uwsgi_cache_min_uses number ; | uwsgi_cache_min_uses 1; | http , server , location |
| `uwsgi_cache_path` | uwsgi_cache_path path [ levels = levels ] [ use_temp_path = on \| off ] keys_zone = name : size [ inactive = time ] [ ma | — | http |
| `uwsgi_cache_use_stale` | uwsgi_cache_use_stale error \| timeout \| invalid_header \| updating \| http_500 \| http_503 \| http_403 \| http_404 \|  | uwsgi_cache_use_stale off; | http , server , location |
| `uwsgi_cache_valid` | uwsgi_cache_valid [ code ...] time ; | — | http , server , location |
| `uwsgi_connect_timeout` | uwsgi_connect_timeout time ; | uwsgi_connect_timeout 60s; | http , server , location |
| `uwsgi_hide_header` | uwsgi_hide_header field ; | — | http , server , location |
| `uwsgi_ignore_client_abort` | uwsgi_ignore_client_abort on \| off ; | uwsgi_ignore_client_abort off; | http , server , location |
| `uwsgi_ignore_headers` | uwsgi_ignore_headers field ...; | — | http , server , location |
| `uwsgi_intercept_errors` | uwsgi_intercept_errors on \| off ; | uwsgi_intercept_errors off; | http , server , location |
| `uwsgi_max_temp_file_size` | uwsgi_max_temp_file_size size ; | uwsgi_max_temp_file_size 1024m; | http , server , location |
| `uwsgi_modifier1` | uwsgi_modifier1 number ; | uwsgi_modifier1 0; | http , server , location |
| `uwsgi_modifier2` | uwsgi_modifier2 number ; | uwsgi_modifier2 0; | http , server , location |
| `uwsgi_next_upstream` | uwsgi_next_upstream error \| timeout \| denied \| invalid_header \| http_500 \| http_503 \| http_403 \| http_404 \| http | uwsgi_next_upstream error timeout; | http , server , location |
| `uwsgi_no_cache` | uwsgi_no_cache string ...; | — | http , server , location |
| `uwsgi_param` | uwsgi_param parameter value [ if_not_empty ]; | uwsgi_param HTTP_HOST $host$is_request_port$request_port; | http , server , location |
| `uwsgi_pass` | uwsgi_pass [ protocol ://] address ; | — | location , if in location |
| `uwsgi_pass_header` | uwsgi_pass_header field ; | — | http , server , location |
| `uwsgi_pass_request_body` | uwsgi_pass_request_body on \| off ; | uwsgi_pass_request_body on; | http , server , location |
| `uwsgi_pass_request_headers` | uwsgi_pass_request_headers on \| off ; | uwsgi_pass_request_headers on; | http , server , location |
| `uwsgi_read_timeout` | uwsgi_read_timeout time ; | uwsgi_read_timeout 60s; | http , server , location |
| `uwsgi_send_timeout` | uwsgi_send_timeout time ; | uwsgi_send_timeout 60s; | http , server , location |
| `uwsgi_store` | uwsgi_store on \| off \| string ; | uwsgi_store off; | http , server , location |
| `uwsgi_store_access` | uwsgi_store_access users : permissions ...; | uwsgi_store_access user:rw; | http , server , location |
| `uwsgi_temp_file_write_size` | uwsgi_temp_file_write_size size ; | uwsgi_temp_file_write_size 8k\|16k; | http , server , location |
| `uwsgi_temp_path` | uwsgi_temp_path path [ level1 [ level2 [ level3 ]]]; | uwsgi_temp_path uwsgi_temp; | http , server , location |

## Directive Details

### `uwsgi_bind`

**Syntax**: `uwsgi_bind address [ transparent ] | off ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Makes outgoing connections to a uwsgi server originate from the specified local IP address with an optional port (1.11.2). Parameter value can contain variables (1.3.12). The special value off (1.3.12) cancels the effect of the uwsgi_bind directive inherited from the previous configuration level, which allows the system to auto-assign the local IP address and port.

### `uwsgi_buffer_size`

**Syntax**: `uwsgi_buffer_size size ;`  
**Default**: `uwsgi_buffer_size 4k|8k;`  
**Context**: `http , server , location`  

Sets the size of the buffer used for reading the first part of the response received from the uwsgi server. This part usually contains a small response header; if it exceeds the buffer size, the response is considered invalid . By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform. It can be made smaller, however.

### `uwsgi_buffering`

**Syntax**: `uwsgi_buffering on | off ;`  
**Default**: `uwsgi_buffering on;`  
**Context**: `http , server , location`  

Enables or disables buffering of responses from the uwsgi server. When buffering is enabled, nginx receives a response from the uwsgi server as soon as possible, saving it into the buffers set by the uwsgi_buffer_size and uwsgi_buffers directives. If the whole response does not fit into memory, a part of it can be saved to a temporary file on the disk. Writing to temporary files is controlled by the uwsgi_max_temp_file_size and uwsgi_temp_file_write_size directives. When buffering is disabled, the response is passed to a client synchronously, immediately as it is received. nginx will not try to read the whole response from the uwsgi server. The maximum size of the data that nginx can receive from the server at a time is set by the uwsgi_buffer_size directive.

### `uwsgi_buffers`

**Syntax**: `uwsgi_buffers number size ;`  
**Default**: `uwsgi_buffers 8 4k|8k;`  
**Context**: `http , server , location`  

Sets the number and size of the buffers used for reading a response from the uwsgi server, for a single connection. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.

### `uwsgi_busy_buffers_size`

**Syntax**: `uwsgi_busy_buffers_size size ;`  
**Default**: `uwsgi_busy_buffers_size 8k|16k;`  
**Context**: `http , server , location`  

When buffering of responses from the uwsgi server is enabled, limits the total size of buffers that can be busy sending a response to the client while the response is not yet fully read. In the meantime, the rest of the buffers can be used for reading the response and, if needed, buffering part of the response to a temporary file. By default, size is limited by the size of two buffers set by the uwsgi_buffer_size and uwsgi_buffers directives.

### `uwsgi_cache`

**Syntax**: `uwsgi_cache zone | off ;`  
**Default**: `uwsgi_cache off;`  
**Context**: `http , server , location`  

Defines a shared memory zone used for caching. The same zone can be used in several places. Parameter value can contain variables (1.7.9). The off parameter disables caching inherited from the previous configuration level.

### `uwsgi_cache_bypass`

**Syntax**: `uwsgi_cache_bypass string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be taken from a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be taken from the cache: uwsgi_cache_bypass $cookie_nocache $arg_nocache$arg_comment; uwsgi_cache_bypass $http_pragma $http_authorization; Can be used along with the uwsgi_no_cache directive.

### `uwsgi_cache_key`

**Syntax**: `uwsgi_cache_key string ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines a key for caching, for example uwsgi_cache_key localhost:9000$request_uri;

### `uwsgi_cache_methods`

**Syntax**: `uwsgi_cache_methods GET | HEAD | POST ...;`  
**Default**: `uwsgi_cache_methods GET HEAD;`  
**Context**: `http , server , location`  

If the client request method is listed in this directive then the response will be cached. “ GET ” and “ HEAD ” methods are always added to the list, though it is recommended to specify them explicitly. See also the uwsgi_no_cache directive.

### `uwsgi_cache_min_uses`

**Syntax**: `uwsgi_cache_min_uses number ;`  
**Default**: `uwsgi_cache_min_uses 1;`  
**Context**: `http , server , location`  

Sets the number of requests after which the response will be cached.

### `uwsgi_cache_path`

**Syntax**: `uwsgi_cache_path path [ levels = levels ] [ use_temp_path = on | off ] keys_zone = name : size [ inactive = time ] [ max_size = size ] [ min_free = size ] [ manager_files = number ] [ manager_sleep = time ] [ manager_threshold = time ] [ loader_files = number ] [ loader_sleep = time ] [ loader_threshold = time ] [ purger = on | off ] [ purger_files = number ] [ purger_sleep = time ] [ purger_threshold = time ];`  
**Default**: `—`  
**Context**: `http`  

Sets the path and other parameters of a cache. Cache data are stored in files. The file name in a cache is a result of applying the MD5 function to the cache key . The levels parameter defines hierarchy levels of a cache: from 1 to 3, each level accepts values 1 or 2. For example, in the following configuration uwsgi_cache_path /data/nginx/cache levels=1:2 keys_zone=one:10m; file names in a cache will look like this: /data/nginx/cache/ c / 29 /b7f54b2df7773722d382f4809d650 29c

### `uwsgi_cache_use_stale`

**Syntax**: `uwsgi_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_503 | http_403 | http_404 | http_429 | off ...;`  
**Default**: `uwsgi_cache_use_stale off;`  
**Context**: `http , server , location`  

Determines in which cases a stale cached response can be used when an error occurs during communication with the uwsgi server. The directive’s parameters match the parameters of the uwsgi_next_upstream directive. The error parameter also permits using a stale cached response if a uwsgi server to process a request cannot be selected.

### `uwsgi_cache_valid`

**Syntax**: `uwsgi_cache_valid [ code ...] time ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets caching time for different response codes. For example, the following directives uwsgi_cache_valid 200 302 10m; uwsgi_cache_valid 404 1m; set 10 minutes of caching for responses with codes 200 and 302 and 1 minute for responses with code 404. If only caching time is specified

### `uwsgi_connect_timeout`

**Syntax**: `uwsgi_connect_timeout time ;`  
**Default**: `uwsgi_connect_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for establishing a connection with a uwsgi server. It should be noted that this timeout cannot usually exceed 75 seconds.

### `uwsgi_hide_header`

**Syntax**: `uwsgi_hide_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

By default, nginx does not pass the header fields “Status” and “X-Accel-...” from the response of a uwsgi server to a client. The uwsgi_hide_header directive sets additional fields that will not be passed. If, on the contrary, the passing of fields needs to be permitted, the uwsgi_pass_header directive can be used.

### `uwsgi_ignore_client_abort`

**Syntax**: `uwsgi_ignore_client_abort on | off ;`  
**Default**: `uwsgi_ignore_client_abort off;`  
**Context**: `http , server , location`  

Determines whether the connection with a uwsgi server should be closed when a client closes the connection without waiting for a response.

### `uwsgi_ignore_headers`

**Syntax**: `uwsgi_ignore_headers field ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Disables processing of certain response header fields from the uwsgi server. The following fields can be ignored: “X-Accel-Redirect”, “X-Accel-Expires”, “X-Accel-Limit-Rate” (1.1.6), “X-Accel-Buffering” (1.1.6), “X-Accel-Charset” (1.1.6), “Expires”, “Cache-Control”, “Set-Cookie” (0.8.44), and “Vary” (1.7.7). If not disabled, processing of these header fields has the following effect:

### `uwsgi_intercept_errors`

**Syntax**: `uwsgi_intercept_errors on | off ;`  
**Default**: `uwsgi_intercept_errors off;`  
**Context**: `http , server , location`  

Determines whether a uwsgi server responses with codes greater than or equal to 300 should be passed to a client or be intercepted and redirected to nginx for processing with the error_page directive.

### `uwsgi_max_temp_file_size`

**Syntax**: `uwsgi_max_temp_file_size size ;`  
**Default**: `uwsgi_max_temp_file_size 1024m;`  
**Context**: `http , server , location`  

When buffering of responses from the uwsgi server is enabled, and the whole response does not fit into the buffers set by the uwsgi_buffer_size and uwsgi_buffers directives, a part of the response can be saved to a temporary file. This directive sets the maximum size of the temporary file. The size of data written to the temporary file at a time is set by the uwsgi_temp_file_write_size directive. The zero value disables buffering of responses to temporary files.

### `uwsgi_modifier1`

**Syntax**: `uwsgi_modifier1 number ;`  
**Default**: `uwsgi_modifier1 0;`  
**Context**: `http , server , location`  

Sets the value of the modifier1 field in the uwsgi packet header .

### `uwsgi_modifier2`

**Syntax**: `uwsgi_modifier2 number ;`  
**Default**: `uwsgi_modifier2 0;`  
**Context**: `http , server , location`  

Sets the value of the modifier2 field in the uwsgi packet header .

### `uwsgi_next_upstream`

**Syntax**: `uwsgi_next_upstream error | timeout | denied | invalid_header | http_500 | http_503 | http_403 | http_404 | http_429 | non_idempotent | off ...;`  
**Default**: `uwsgi_next_upstream error timeout;`  
**Context**: `http , server , location`  

Specifies in which cases a request should be passed to the next server:

### `uwsgi_no_cache`

**Syntax**: `uwsgi_no_cache string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be saved to a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be saved: uwsgi_no_cache $cookie_nocache $arg_nocache$arg_comment; uwsgi_no_cache $http_pragma $http_authorization; Can be used along with the uwsgi_cache_bypass directive.

### `uwsgi_param`

**Syntax**: `uwsgi_param parameter value [ if_not_empty ];`  
**Default**: `uwsgi_param HTTP_HOST $host$is_request_port$request_port;`  
**Context**: `http , server , location`  

Sets a parameter that should be passed to the uwsgi server. The value can contain text, variables, and their combination. These directives are inherited from the previous configuration level if and only if there are no uwsgi_param directives defined on the current level. Standard CGI environment variables should be provided as uwsgi headers, see the uwsgi_params file provided in the distribution: location / { include uwsgi_params; ... }

### `uwsgi_pass`

**Syntax**: `uwsgi_pass [ protocol ://] address ;`  
**Default**: `—`  
**Context**: `location , if in location`  

Sets the protocol and address of a uwsgi server. As a protocol , “ uwsgi ” or “ suwsgi ” (secured uwsgi, uwsgi over SSL) can be specified. The address can be specified as a domain name or IP address, and a port: uwsgi_pass localhost:9000; uwsgi_pass uwsgi://localhost:9000; uwsgi_pass suwsgi://[2001:db8::1]:9090; or as a UNIX-domain socket path: uwsgi_pass unix:/tmp/uwsgi.socket;

### `uwsgi_pass_header`

**Syntax**: `uwsgi_pass_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Permits passing otherwise disabled header fields from a uwsgi server to a client.

### `uwsgi_pass_request_body`

**Syntax**: `uwsgi_pass_request_body on | off ;`  
**Default**: `uwsgi_pass_request_body on;`  
**Context**: `http , server , location`  

Indicates whether the original request body is passed to the uwsgi server. See also the uwsgi_pass_request_headers directive.

### `uwsgi_pass_request_headers`

**Syntax**: `uwsgi_pass_request_headers on | off ;`  
**Default**: `uwsgi_pass_request_headers on;`  
**Context**: `http , server , location`  

Indicates whether the header fields of the original request are passed to the uwsgi server. See also the uwsgi_pass_request_body directive.

### `uwsgi_read_timeout`

**Syntax**: `uwsgi_read_timeout time ;`  
**Default**: `uwsgi_read_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading a response from the uwsgi server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the uwsgi server does not transmit anything within this time, the connection is closed.

### `uwsgi_send_timeout`

**Syntax**: `uwsgi_send_timeout time ;`  
**Default**: `uwsgi_send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a request to the uwsgi server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the uwsgi server does not receive anything within this time, the connection is closed.

### `uwsgi_store`

**Syntax**: `uwsgi_store on | off | string ;`  
**Default**: `uwsgi_store off;`  
**Context**: `http , server , location`  

Enables saving of files to a disk. The on parameter saves files with paths corresponding to the directives alias or root . The off parameter disables saving of files. In addition, the file name can be set explicitly using the string with variables: uwsgi_store /data/www$original_uri; The modification time of files is set according to the received “Last-Modified” response header field. The response is first written to a temporary file, and then the file is renamed. Starting from version 0.8.9, temporary files and the persistent store can be put on different file systems. However, be aware that in this case a file is copied across two file systems instead of the cheap renaming operation. It is thus recommended that for any given location both saved files and a directory holding temporary files, set by the uwsgi_temp_path directive, are put on the same file system.

### `uwsgi_store_access`

**Syntax**: `uwsgi_store_access users : permissions ...;`  
**Default**: `uwsgi_store_access user:rw;`  
**Context**: `http , server , location`  

Sets access permissions for newly created files and directories, e.g.: uwsgi_store_access user:rw group:rw all:r; If any group or all access permissions are specified then user permissions may be omitted:

### `uwsgi_temp_file_write_size`

**Syntax**: `uwsgi_temp_file_write_size size ;`  
**Default**: `uwsgi_temp_file_write_size 8k|16k;`  
**Context**: `http , server , location`  

Limits the size of data written to a temporary file at a time, when buffering of responses from the uwsgi server to temporary files is enabled. By default, size is limited by two buffers set by the uwsgi_buffer_size and uwsgi_buffers directives. The maximum size of a temporary file is set by the uwsgi_max_temp_file_size directive.

### `uwsgi_temp_path`

**Syntax**: `uwsgi_temp_path path [ level1 [ level2 [ level3 ]]];`  
**Default**: `uwsgi_temp_path uwsgi_temp;`  
**Context**: `http , server , location`  

Defines a directory for storing temporary files with data received from uwsgi servers. Up to three-level subdirectory hierarchy can be used underneath the specified directory. For example, in the following configuration uwsgi_temp_path /spool/nginx/uwsgi_temp 1 2; a temporary file might look like this: /spool/nginx/uwsgi_temp/ 7 / 45 /00000123 457
