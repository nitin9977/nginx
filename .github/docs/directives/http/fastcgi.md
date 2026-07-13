# Module ngx_http_fastcgi_module

**Source**: https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives fastcgi_allow_upstream fastcgi_bind fastcgi_bind_dynamic fastcgi_buffer_size fastcgi_buffering fastcgi_buffers fastcgi_busy_buffers_size fastcgi_cache fastcgi_cache_background_update fastcgi_cache_bypass fastcgi_cache_key fastcgi_cache_lock fastcgi_cache_lock_age fastcgi_cache_lock_timeout fastcgi_cache_max_range_offset fastcgi_cache_methods fastcgi_cache_min_uses fastcgi_cache_path fastcgi_cache_purge fastcgi_cache_revalidate fastcgi_cache_use_stale fastcgi_cache_valid fastcgi_catch_stderr fastcgi_connect_timeout fastcgi_force_ranges fastcgi_hide_header fastcgi_ignore_client_abort fastcgi_ignore_headers fastcgi_index fastcgi_intercept_errors fastcgi_keep_conn fastcgi_limit_rate fastcgi_max_temp_file_size fastcgi_next_upstream fastcgi_next_upstream_timeout 

## Example Configuration

```nginx
location / {
    fastcgi_pass  localhost:9000;
    fastcgi_index index.php;

    fastcgi_param SCRIPT_FILENAME /home/www/scripts/php$fastcgi_script_name;
    fastcgi_param QUERY_STRING    $query_string;
    fastcgi_param REQUEST_METHOD  $request_method;
    fastcgi_param CONTENT_TYPE    $content_type;
    fastcgi_param CONTENT_LENGTH  $content_length;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `fastcgi_buffer_size` | fastcgi_buffer_size size ; | fastcgi_buffer_size 4k\|8k; | http , server , location |
| `fastcgi_buffers` | fastcgi_buffers number size ; | fastcgi_buffers 8 4k\|8k; | http , server , location |
| `fastcgi_busy_buffers_size` | fastcgi_busy_buffers_size size ; | fastcgi_busy_buffers_size 8k\|16k; | http , server , location |
| `fastcgi_cache` | fastcgi_cache zone \| off ; | fastcgi_cache off; | http , server , location |
| `fastcgi_cache_bypass` | fastcgi_cache_bypass string ...; | — | http , server , location |
| `fastcgi_cache_key` | fastcgi_cache_key string ; | — | http , server , location |
| `fastcgi_cache_min_uses` | fastcgi_cache_min_uses number ; | fastcgi_cache_min_uses 1; | http , server , location |
| `fastcgi_cache_path` | fastcgi_cache_path path [ levels = levels ] [ use_temp_path = on \| off ] keys_zone = name : size [ inactive = time ] [  | — | http |
| `fastcgi_cache_use_stale` | fastcgi_cache_use_stale error \| timeout \| invalid_header \| updating \| http_500 \| http_503 \| http_403 \| http_404 \ | fastcgi_cache_use_stale off; | http , server , location |
| `fastcgi_cache_valid` | fastcgi_cache_valid [ code ...] time ; | — | http , server , location |
| `fastcgi_catch_stderr` | fastcgi_catch_stderr string ; | — | http , server , location |
| `fastcgi_connect_timeout` | fastcgi_connect_timeout time ; | fastcgi_connect_timeout 60s; | http , server , location |
| `fastcgi_hide_header` | fastcgi_hide_header field ; | — | http , server , location |
| `fastcgi_ignore_client_abort` | fastcgi_ignore_client_abort on \| off ; | fastcgi_ignore_client_abort off; | http , server , location |
| `fastcgi_ignore_headers` | fastcgi_ignore_headers field ...; | — | http , server , location |
| `fastcgi_index` | fastcgi_index name ; | — | http , server , location |
| `fastcgi_intercept_errors` | fastcgi_intercept_errors on \| off ; | fastcgi_intercept_errors off; | http , server , location |
| `fastcgi_max_temp_file_size` | fastcgi_max_temp_file_size size ; | fastcgi_max_temp_file_size 1024m; | http , server , location |
| `fastcgi_next_upstream` | fastcgi_next_upstream error \| timeout \| denied \| invalid_header \| http_500 \| http_503 \| http_403 \| http_404 \| ht | fastcgi_next_upstream error timeout; | http , server , location |
| `fastcgi_no_cache` | fastcgi_no_cache string ...; | — | http , server , location |
| `fastcgi_param` | fastcgi_param parameter value [ if_not_empty ]; | fastcgi_param HTTP_HOST $host$is_request_port$request_port; | http , server , location |
| `fastcgi_pass` | fastcgi_pass address ; | — | location , if in location |
| `fastcgi_pass_header` | fastcgi_pass_header field ; | — | http , server , location |
| `fastcgi_pass_request_body` | fastcgi_pass_request_body on \| off ; | fastcgi_pass_request_body on; | http , server , location |
| `fastcgi_pass_request_headers` | fastcgi_pass_request_headers on \| off ; | fastcgi_pass_request_headers on; | http , server , location |
| `fastcgi_read_timeout` | fastcgi_read_timeout time ; | fastcgi_read_timeout 60s; | http , server , location |
| `fastcgi_send_lowat` | fastcgi_send_lowat size ; | fastcgi_send_lowat 0; | http , server , location |
| `fastcgi_send_timeout` | fastcgi_send_timeout time ; | fastcgi_send_timeout 60s; | http , server , location |
| `fastcgi_split_path_info` | fastcgi_split_path_info regex ; | — | location |
| `fastcgi_store` | fastcgi_store on \| off \| string ; | fastcgi_store off; | http , server , location |
| `fastcgi_store_access` | fastcgi_store_access users : permissions ...; | fastcgi_store_access user:rw; | http , server , location |
| `fastcgi_temp_file_write_size` | fastcgi_temp_file_write_size size ; | fastcgi_temp_file_write_size 8k\|16k; | http , server , location |
| `fastcgi_temp_path` | fastcgi_temp_path path [ level1 [ level2 [ level3 ]]]; | fastcgi_temp_path fastcgi_temp; | http , server , location |

## Directive Details

### `fastcgi_buffer_size`

**Syntax**: `fastcgi_buffer_size size ;`  
**Default**: `fastcgi_buffer_size 4k|8k;`  
**Context**: `http , server , location`  

Sets the size of the buffer used for reading the first part of the response received from the FastCGI server. This part usually contains a small response header; if it exceeds the buffer size, the response is considered invalid . By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform. It can be made smaller, however.

### `fastcgi_buffers`

**Syntax**: `fastcgi_buffers number size ;`  
**Default**: `fastcgi_buffers 8 4k|8k;`  
**Context**: `http , server , location`  

Sets the number and size of the buffers used for reading a response from the FastCGI server, for a single connection. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.

### `fastcgi_busy_buffers_size`

**Syntax**: `fastcgi_busy_buffers_size size ;`  
**Default**: `fastcgi_busy_buffers_size 8k|16k;`  
**Context**: `http , server , location`  

When buffering of responses from the FastCGI server is enabled, limits the total size of buffers that can be busy sending a response to the client while the response is not yet fully read. In the meantime, the rest of the buffers can be used for reading the response and, if needed, buffering part of the response to a temporary file. By default, size is limited by the size of two buffers set by the fastcgi_buffer_size and fastcgi_buffers directives.

### `fastcgi_cache`

**Syntax**: `fastcgi_cache zone | off ;`  
**Default**: `fastcgi_cache off;`  
**Context**: `http , server , location`  

Defines a shared memory zone used for caching. The same zone can be used in several places. Parameter value can contain variables (1.7.9). The off parameter disables caching inherited from the previous configuration level.

### `fastcgi_cache_bypass`

**Syntax**: `fastcgi_cache_bypass string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be taken from a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be taken from the cache: fastcgi_cache_bypass $cookie_nocache $arg_nocache$arg_comment; fastcgi_cache_bypass $http_pragma $http_authorization; Can be used along with the fastcgi_no_cache directive.

### `fastcgi_cache_key`

**Syntax**: `fastcgi_cache_key string ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines a key for caching, for example fastcgi_cache_key localhost:9000$request_uri;

### `fastcgi_cache_min_uses`

**Syntax**: `fastcgi_cache_min_uses number ;`  
**Default**: `fastcgi_cache_min_uses 1;`  
**Context**: `http , server , location`  

Sets the number of requests after which the response will be cached.

### `fastcgi_cache_path`

**Syntax**: `fastcgi_cache_path path [ levels = levels ] [ use_temp_path = on | off ] keys_zone = name : size [ inactive = time ] [ max_size = size ] [ min_free = size ] [ manager_files = number ] [ manager_sleep = time ] [ manager_threshold = time ] [ loader_files = number ] [ loader_sleep = time ] [ loader_threshold = time ] [ purger = on | off ] [ purger_files = number ] [ purger_sleep = time ] [ purger_threshold = time ];`  
**Default**: `—`  
**Context**: `http`  

Sets the path and other parameters of a cache. Cache data are stored in files. Both the key and file name in a cache are a result of applying the MD5 function to the proxied URL. The levels parameter defines hierarchy levels of a cache: from 1 to 3, each level accepts values 1 or 2. For example, in the following configuration fastcgi_cache_path /data/nginx/cache levels=1:2 keys_zone=one:10m; file names in a cache will look like this: /data/nginx/cache/ c / 29 /b7f54b2df7773722d382f4809d650 29c

### `fastcgi_cache_use_stale`

**Syntax**: `fastcgi_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_503 | http_403 | http_404 | http_429 | off ...;`  
**Default**: `fastcgi_cache_use_stale off;`  
**Context**: `http , server , location`  

Determines in which cases a stale cached response can be used when an error occurs during communication with the FastCGI server. The directive’s parameters match the parameters of the fastcgi_next_upstream directive. The error parameter also permits using a stale cached response if a FastCGI server to process a request cannot be selected.

### `fastcgi_cache_valid`

**Syntax**: `fastcgi_cache_valid [ code ...] time ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets caching time for different response codes. For example, the following directives fastcgi_cache_valid 200 302 10m; fastcgi_cache_valid 404 1m; set 10 minutes of caching for responses with codes 200 and 302 and 1 minute for responses with code 404. If only caching time is specified

### `fastcgi_catch_stderr`

**Syntax**: `fastcgi_catch_stderr string ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets a string to search for in the error stream of a response received from a FastCGI server. If the string is found then it is considered that the FastCGI server has returned an invalid response . This allows handling application errors in nginx, for example: location /php/ { fastcgi_pass backend:9000; ... fastcgi_catch_stderr "PHP Fatal error"; fastcgi_next_upstream error timeout invalid_header; }

### `fastcgi_connect_timeout`

**Syntax**: `fastcgi_connect_timeout time ;`  
**Default**: `fastcgi_connect_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for establishing a connection with a FastCGI server. It should be noted that this timeout cannot usually exceed 75 seconds.

### `fastcgi_hide_header`

**Syntax**: `fastcgi_hide_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

By default, nginx does not pass the header fields “Status” and “X-Accel-...” from the response of a FastCGI server to a client. The fastcgi_hide_header directive sets additional fields that will not be passed. If, on the contrary, the passing of fields needs to be permitted, the fastcgi_pass_header directive can be used.

### `fastcgi_ignore_client_abort`

**Syntax**: `fastcgi_ignore_client_abort on | off ;`  
**Default**: `fastcgi_ignore_client_abort off;`  
**Context**: `http , server , location`  

Determines whether the connection with a FastCGI server should be closed when a client closes the connection without waiting for a response.

### `fastcgi_ignore_headers`

**Syntax**: `fastcgi_ignore_headers field ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Disables processing of certain response header fields from the FastCGI server. The following fields can be ignored: “X-Accel-Redirect”, “X-Accel-Expires”, “X-Accel-Limit-Rate” (1.1.6), “X-Accel-Buffering” (1.1.6), “X-Accel-Charset” (1.1.6), “Expires”, “Cache-Control”, “Set-Cookie” (0.8.44), and “Vary” (1.7.7). If not disabled, processing of these header fields has the following effect:

### `fastcgi_index`

**Syntax**: `fastcgi_index name ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Sets a file name that will be appended after a URI that ends with a slash, in the value of the $fastcgi_script_name variable. For example, with these settings fastcgi_index index.php; fastcgi_param SCRIPT_FILENAME /home/www/scripts/php$fastcgi_script_name; and the “ /page.php ” request, the SCRIPT_FILENAME parameter will be equal to “ /home/www/scripts/php/page.php ”, and with the “ / ” request it will be equal to “ /home/www/scripts/php/index.php ”.

### `fastcgi_intercept_errors`

**Syntax**: `fastcgi_intercept_errors on | off ;`  
**Default**: `fastcgi_intercept_errors off;`  
**Context**: `http , server , location`  

Determines whether FastCGI server responses with codes greater than or equal to 300 should be passed to a client or be intercepted and redirected to nginx for processing with the error_page directive.

### `fastcgi_max_temp_file_size`

**Syntax**: `fastcgi_max_temp_file_size size ;`  
**Default**: `fastcgi_max_temp_file_size 1024m;`  
**Context**: `http , server , location`  

When buffering of responses from the FastCGI server is enabled, and the whole response does not fit into the buffers set by the fastcgi_buffer_size and fastcgi_buffers directives, a part of the response can be saved to a temporary file. This directive sets the maximum size of the temporary file. The size of data written to the temporary file at a time is set by the fastcgi_temp_file_write_size directive. The zero value disables buffering of responses to temporary files.

### `fastcgi_next_upstream`

**Syntax**: `fastcgi_next_upstream error | timeout | denied | invalid_header | http_500 | http_503 | http_403 | http_404 | http_429 | non_idempotent | off ...;`  
**Default**: `fastcgi_next_upstream error timeout;`  
**Context**: `http , server , location`  

Specifies in which cases a request should be passed to the next server:

### `fastcgi_no_cache`

**Syntax**: `fastcgi_no_cache string ...;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines conditions under which the response will not be saved to a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be saved: fastcgi_no_cache $cookie_nocache $arg_nocache$arg_comment; fastcgi_no_cache $http_pragma $http_authorization; Can be used along with the fastcgi_cache_bypass directive.

### `fastcgi_param`

**Syntax**: `fastcgi_param parameter value [ if_not_empty ];`  
**Default**: `fastcgi_param HTTP_HOST $host$is_request_port$request_port;`  
**Context**: `http , server , location`  

Sets a parameter that should be passed to the FastCGI server. The value can contain text, variables, and their combination. These directives are inherited from the previous configuration level if and only if there are no fastcgi_param directives defined on the current level. The following example shows the minimum required settings for PHP: fastcgi_param SCRIPT_FILENAME /home/www/scripts/php$fastcgi_script_name; fastcgi_param QUERY_STRING $query_string;

### `fastcgi_pass`

**Syntax**: `fastcgi_pass address ;`  
**Default**: `—`  
**Context**: `location , if in location`  

Sets the address of a FastCGI server. The address can be specified as a domain name or IP address, and a port: fastcgi_pass localhost:9000; or as a UNIX-domain socket path: fastcgi_pass unix:/tmp/fastcgi.socket;

### `fastcgi_pass_header`

**Syntax**: `fastcgi_pass_header field ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Permits passing otherwise disabled header fields from a FastCGI server to a client.

### `fastcgi_pass_request_body`

**Syntax**: `fastcgi_pass_request_body on | off ;`  
**Default**: `fastcgi_pass_request_body on;`  
**Context**: `http , server , location`  

Indicates whether the original request body is passed to the FastCGI server. See also the fastcgi_pass_request_headers directive.

### `fastcgi_pass_request_headers`

**Syntax**: `fastcgi_pass_request_headers on | off ;`  
**Default**: `fastcgi_pass_request_headers on;`  
**Context**: `http , server , location`  

Indicates whether the header fields of the original request are passed to the FastCGI server. See also the fastcgi_pass_request_body directive.

### `fastcgi_read_timeout`

**Syntax**: `fastcgi_read_timeout time ;`  
**Default**: `fastcgi_read_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading a response from the FastCGI server. The timeout is set only between two successive read operations, not for the transmission of the whole response. If the FastCGI server does not transmit anything within this time, the connection is closed.

### `fastcgi_send_lowat`

**Syntax**: `fastcgi_send_lowat size ;`  
**Default**: `fastcgi_send_lowat 0;`  
**Context**: `http , server , location`  

If the directive is set to a non-zero value, nginx will try to minimize the number of send operations on outgoing connections to a FastCGI server by using either NOTE_LOWAT flag of the kqueue method, or the SO_SNDLOWAT socket option, with the specified size . This directive is ignored on Linux, Solaris, and Windows.

### `fastcgi_send_timeout`

**Syntax**: `fastcgi_send_timeout time ;`  
**Default**: `fastcgi_send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a request to the FastCGI server. The timeout is set only between two successive write operations, not for the transmission of the whole request. If the FastCGI server does not receive anything within this time, the connection is closed.

### `fastcgi_split_path_info`

**Syntax**: `fastcgi_split_path_info regex ;`  
**Default**: `—`  
**Context**: `location`  

Defines a regular expression that captures a value for the $fastcgi_path_info variable. The regular expression should have two captures: the first becomes a value of the $fastcgi_script_name variable, the second becomes a value of the $fastcgi_path_info variable. For example, with these settings location ~ ^(.+\.php)(.*)$ { fastcgi_split_path_info ^(.+\.php)(.*)$; fastcgi_param SCRIPT_FILENAME /path/to/php$fastcgi_script_name; fastcgi_param PATH_INFO $fastcgi_path_info; and the “ /show.php/article/0001 ” request, the SCRIPT_FILENAME parameter will be equal to “ /path/to/php/show.php ”, and the PATH_INFO parameter will be equal to “ /article/0001 ”.

### `fastcgi_store`

**Syntax**: `fastcgi_store on | off | string ;`  
**Default**: `fastcgi_store off;`  
**Context**: `http , server , location`  

Enables saving of files to a disk. The on parameter saves files with paths corresponding to the directives alias or root . The off parameter disables saving of files. In addition, the file name can be set explicitly using the string with variables: fastcgi_store /data/www$original_uri; The modification time of files is set according to the received “Last-Modified” response header field. The response is first written to a temporary file, and then the file is renamed. Starting from version 0.8.9, temporary files and the persistent store can be put on different file systems. However, be aware that in this case a file is copied across two file systems instead of the cheap renaming operation. It is thus recommended that for any given location both saved files and a directory holding temporary files, set by the fastcgi_temp_path directive, are put on the same file system.

### `fastcgi_store_access`

**Syntax**: `fastcgi_store_access users : permissions ...;`  
**Default**: `fastcgi_store_access user:rw;`  
**Context**: `http , server , location`  

Sets access permissions for newly created files and directories, e.g.: fastcgi_store_access user:rw group:rw all:r; If any group or all access permissions are specified then user permissions may be omitted:

### `fastcgi_temp_file_write_size`

**Syntax**: `fastcgi_temp_file_write_size size ;`  
**Default**: `fastcgi_temp_file_write_size 8k|16k;`  
**Context**: `http , server , location`  

Limits the size of data written to a temporary file at a time, when buffering of responses from the FastCGI server to temporary files is enabled. By default, size is limited by two buffers set by the fastcgi_buffer_size and fastcgi_buffers directives. The maximum size of a temporary file is set by the fastcgi_max_temp_file_size directive.

### `fastcgi_temp_path`

**Syntax**: `fastcgi_temp_path path [ level1 [ level2 [ level3 ]]];`  
**Default**: `fastcgi_temp_path fastcgi_temp;`  
**Context**: `http , server , location`  

Defines a directory for storing temporary files with data received from FastCGI servers. Up to three-level subdirectory hierarchy can be used underneath the specified directory. For example, in the following configuration fastcgi_temp_path /spool/nginx/fastcgi_temp 1 2; a temporary file might look like this: /spool/nginx/fastcgi_temp/ 7 / 45 /00000123 457
