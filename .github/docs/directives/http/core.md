# Module ngx_http_core_module

**Source**: https://nginx.org/en/docs/http/ngx_http_core_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Directives absolute_redirect aio aio_write alias auth_delay chunked_transfer_encoding client_body_buffer_size client_body_in_file_only client_body_in_single_buffer client_body_temp_path client_body_timeout client_header_buffer_size client_header_timeout client_max_body_size connection_pool_size default_type directio directio_alignment disable_symlinks early_hints error_log_tag error_page etag http if_modified_since ignore_invalid_headers internal keepalive_disable keepalive_min_timeout keepalive_requests keepalive_time keepalive_timeout large_client_header_buffers limit_except limit_rate limit_rate_after lingering_close lingering_time lingering_timeout listen location log_not_found log_subrequest max_headers max_ranges merge_slashes msie_padding msie_refresh open_file_cache open_file_cache

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `alias` | alias path ; | — | location |
| `chunked_transfer_encoding` | chunked_transfer_encoding on \| off ; | chunked_transfer_encoding on; | http , server , location |
| `client_body_buffer_size` | client_body_buffer_size size ; | client_body_buffer_size 8k\|16k; | http , server , location |
| `client_body_in_file_only` | client_body_in_file_only on \| clean \| off ; | client_body_in_file_only off; | http , server , location |
| `client_body_in_single_buffer` | client_body_in_single_buffer on \| off ; | client_body_in_single_buffer off; | http , server , location |
| `client_body_temp_path` | client_body_temp_path path [ level1 [ level2 [ level3 ]]]; | client_body_temp_path client_body_temp; | http , server , location |
| `client_body_timeout` | client_body_timeout time ; | client_body_timeout 60s; | http , server , location |
| `client_header_buffer_size` | client_header_buffer_size size ; | client_header_buffer_size 1k; | http , server |
| `client_header_timeout` | client_header_timeout time ; | client_header_timeout 60s; | http , server |
| `client_max_body_size` | client_max_body_size size ; | client_max_body_size 1m; | http , server , location |
| `connection_pool_size` | connection_pool_size size ; | connection_pool_size 256\|512; | http , server |
| `default_type` | default_type mime-type ; | default_type text/plain; | http , server , location |
| `error_page` | error_page code ... [ = [ response ]] uri ; | — | http , server , location , if in location |
| `http` | http { ... } | — | main |
| `ignore_invalid_headers` | ignore_invalid_headers on \| off ; | ignore_invalid_headers on; | http , server |
| `internal` | internal ; | — | location |
| `keepalive_disable` | keepalive_disable none \| browser ...; | keepalive_disable msie6; | http , server , location |
| `keepalive_timeout` | keepalive_timeout timeout [ header_timeout ]; | keepalive_timeout 75s; | http , server , location |
| `large_client_header_buffers` | large_client_header_buffers number size ; | large_client_header_buffers 4 8k; | http , server |
| `limit_except` | limit_except method ... { ... } | — | location |
| `limit_rate` | limit_rate rate ; | limit_rate 0; | http , server , location , if in location |
| `lingering_time` | lingering_time time ; | lingering_time 30s; | http , server , location |
| `lingering_timeout` | lingering_timeout time ; | lingering_timeout 5s; | http , server , location |
| `listen` | listen address [: port ] [ default_server ] [ ssl ] [ http2 \| quic ] [ proxy_protocol ] [ setfib = number ] [ fastopen  | listen *:80 \| *:8000; | server |
| `location` | location [ = \| ~ \| ~* \| ^~ ] uri { ... } location @ name { ... } | — | server , location |
| `log_not_found` | log_not_found on \| off ; | log_not_found on; | http , server , location |
| `log_subrequest` | log_subrequest on \| off ; | log_subrequest off; | http , server , location |
| `merge_slashes` | merge_slashes on \| off ; | merge_slashes on; | http , server |
| `msie_padding` | msie_padding on \| off ; | msie_padding on; | http , server , location |
| `msie_refresh` | msie_refresh on \| off ; | msie_refresh off; | http , server , location |
| `open_file_cache` | open_file_cache off ; open_file_cache max = N [ inactive = time ]; | open_file_cache off; | http , server , location |
| `open_file_cache_errors` | open_file_cache_errors on \| off ; | open_file_cache_errors off; | http , server , location |
| `open_file_cache_min_uses` | open_file_cache_min_uses number ; | open_file_cache_min_uses 1; | http , server , location |
| `open_file_cache_valid` | open_file_cache_valid time ; | open_file_cache_valid 60s; | http , server , location |
| `output_buffers` | output_buffers number size ; | output_buffers 2 32k; | http , server , location |
| `port_in_redirect` | port_in_redirect on \| off ; | port_in_redirect on; | http , server , location |
| `postpone_output` | postpone_output size ; | postpone_output 1460; | http , server , location |
| `read_ahead` | read_ahead size ; | read_ahead 0; | http , server , location |
| `recursive_error_pages` | recursive_error_pages on \| off ; | recursive_error_pages off; | http , server , location |
| `request_pool_size` | request_pool_size size ; | request_pool_size 4k; | http , server |
| `reset_timedout_connection` | reset_timedout_connection on \| off ; | reset_timedout_connection off; | http , server , location |
| `resolver` | resolver address ... [ valid = time ] [ ipv4 = on \| off ] [ ipv6 = on \| off ] [ status_zone = zone ]; | — | http , server , location |
| `resolver_timeout` | resolver_timeout time ; | resolver_timeout 30s; | http , server , location |
| `root` | root path ; | root html; | http , server , location , if in location |
| `satisfy` | satisfy all \| any ; | satisfy all; | http , server , location |
| `send_lowat` | send_lowat size ; | send_lowat 0; | http , server , location |
| `send_timeout` | send_timeout time ; | send_timeout 60s; | http , server , location |
| `sendfile` | sendfile on \| off ; | sendfile off; | http , server , location , if in location |
| `sendfile_max_chunk` | sendfile_max_chunk size ; | sendfile_max_chunk 2m; | http , server , location |
| `server` | server { ... } | — | http |
| `server_name` | server_name name ...; | server_name ""; | server |
| `server_name_in_redirect` | server_name_in_redirect on \| off ; | server_name_in_redirect off; | http , server , location |
| `server_names_hash_bucket_size` | server_names_hash_bucket_size size ; | server_names_hash_bucket_size 32\|64\|128; | http |
| `server_names_hash_max_size` | server_names_hash_max_size size ; | server_names_hash_max_size 512; | http |
| `server_tokens` | server_tokens on \| off \| build \| string ; | server_tokens on; | http , server , location |
| `tcp_nodelay` | tcp_nodelay on \| off ; | tcp_nodelay on; | http , server , location |
| `tcp_nopush` | tcp_nopush on \| off ; | tcp_nopush off; | http , server , location |
| `try_files` | try_files file ... uri ; try_files file ... = code ; | — | server , location |
| `types` | types { ... } | types { text/html html; image/gif gif; image/jpeg jpg; } | http , server , location |
| `types_hash_bucket_size` | types_hash_bucket_size size ; | types_hash_bucket_size 64; | http , server , location |
| `types_hash_max_size` | types_hash_max_size size ; | types_hash_max_size 1024; | http , server , location |
| `underscores_in_headers` | underscores_in_headers on \| off ; | underscores_in_headers off; | http , server |
| `variables_hash_bucket_size` | variables_hash_bucket_size size ; | variables_hash_bucket_size 64; | http |
| `variables_hash_max_size` | variables_hash_max_size size ; | variables_hash_max_size 1024; | http |

## Directive Details

### `alias`

**Syntax**: `alias path ;`  
**Default**: `—`  
**Context**: `location`  

Defines a replacement for the specified location. For example, with the following configuration location /i/ { alias /data/w3/images/; } on request of “ /i/top.gif ”, the file /data/w3/images/top.gif will be sent. The path value can contain variables, except $document_root and $realpath_root .

### `chunked_transfer_encoding`

**Syntax**: `chunked_transfer_encoding on | off ;`  
**Default**: `chunked_transfer_encoding on;`  
**Context**: `http , server , location`  

Allows disabling chunked transfer encoding in HTTP/1.1. It may come in handy when using a software failing to support chunked encoding despite the standard’s requirement.

### `client_body_buffer_size`

**Syntax**: `client_body_buffer_size size ;`  
**Default**: `client_body_buffer_size 8k|16k;`  
**Context**: `http , server , location`  

Sets buffer size for reading client request body. In case the request body is larger than the buffer, the whole body or only its part is written to a temporary file . By default, buffer size is equal to two memory pages. This is 8K on x86, other 32-bit platforms, and x86-64. It is usually 16K on other 64-bit platforms.

### `client_body_in_file_only`

**Syntax**: `client_body_in_file_only on | clean | off ;`  
**Default**: `client_body_in_file_only off;`  
**Context**: `http , server , location`  

Determines whether nginx should save the entire client request body into a file. This directive can be used during debugging, or when using the $request_body_file variable, or the $r->request_body_file method of the module ngx_http_perl_module . When set to the value on , temporary files are not removed after request processing. The value clean will cause the temporary files left after request processing to be removed.

### `client_body_in_single_buffer`

**Syntax**: `client_body_in_single_buffer on | off ;`  
**Default**: `client_body_in_single_buffer off;`  
**Context**: `http , server , location`  

Determines whether nginx should save the entire client request body in a single buffer. The directive is recommended when using the $request_body variable, to save the number of copy operations involved.

### `client_body_temp_path`

**Syntax**: `client_body_temp_path path [ level1 [ level2 [ level3 ]]];`  
**Default**: `client_body_temp_path client_body_temp;`  
**Context**: `http , server , location`  

Defines a directory for storing temporary files holding client request bodies. Up to three-level subdirectory hierarchy can be used under the specified directory. For example, in the following configuration client_body_temp_path /spool/nginx/client_temp 1 2; a path to a temporary file might look like this: /spool/nginx/client_temp/7/45/00000123457

### `client_body_timeout`

**Syntax**: `client_body_timeout time ;`  
**Default**: `client_body_timeout 60s;`  
**Context**: `http , server , location`  

Defines a timeout for reading client request body. The timeout is set only for a period between two successive read operations, not for the transmission of the whole request body. If a client does not transmit anything within this time, the request is terminated with the 408 (Request Time-out) error.

### `client_header_buffer_size`

**Syntax**: `client_header_buffer_size size ;`  
**Default**: `client_header_buffer_size 1k;`  
**Context**: `http , server`  

Sets buffer size for reading client request header. For most requests, a buffer of 1K bytes is enough. However, if a request includes long cookies, or comes from a WAP client, it may not fit into 1K. If a request line or a request header field does not fit into this buffer then larger buffers, configured by the large_client_header_buffers directive, are allocated. If the directive is specified on the server level, the value from the default server can be used. Details are provided in the “ Virtual server selection ” section.

### `client_header_timeout`

**Syntax**: `client_header_timeout time ;`  
**Default**: `client_header_timeout 60s;`  
**Context**: `http , server`  

Defines a timeout for reading client request header. If a client does not transmit the entire header within this time, the request is terminated with the 408 (Request Time-out) error.

### `client_max_body_size`

**Syntax**: `client_max_body_size size ;`  
**Default**: `client_max_body_size 1m;`  
**Context**: `http , server , location`  

Sets the maximum allowed size of the client request body. If the size in a request exceeds the configured value, the 413 (Request Entity Too Large) error is returned to the client. Please be aware that browsers cannot correctly display this error. Setting size to 0 disables checking of client request body size.

### `connection_pool_size`

**Syntax**: `connection_pool_size size ;`  
**Default**: `connection_pool_size 256|512;`  
**Context**: `http , server`  

Allows accurate tuning of per-connection memory allocations. This directive has minimal impact on performance and should not generally be used. By default, the size is equal to 256 bytes on 32-bit platforms and 512 bytes on 64-bit platforms.

### `default_type`

**Syntax**: `default_type mime-type ;`  
**Default**: `default_type text/plain;`  
**Context**: `http , server , location`  

Defines the default MIME type of a response. Mapping of file name extensions to MIME types can be set with the types directive.

### `error_page`

**Syntax**: `error_page code ... [ = [ response ]] uri ;`  
**Default**: `—`  
**Context**: `http , server , location , if in location`  

Defines the URI that will be shown for the specified errors. A uri value can contain variables. Example: error_page 404 /404.html; error_page 500 502 503 504 /50x.html;

### `http`

**Syntax**: `http { ... }`  
**Default**: `—`  
**Context**: `main`  

Provides the configuration file context in which the HTTP server directives are specified.

### `ignore_invalid_headers`

**Syntax**: `ignore_invalid_headers on | off ;`  
**Default**: `ignore_invalid_headers on;`  
**Context**: `http , server`  

Controls whether header fields with invalid names should be ignored. Valid names are composed of English letters, digits, hyphens, and possibly underscores (as controlled by the underscores_in_headers directive). If the directive is specified on the server level, the value from the default server can be used. Details are provided in the “ Virtual server selection ” section.

### `internal`

**Syntax**: `internal ;`  
**Default**: `—`  
**Context**: `location`  

Specifies that a given location can only be used for internal requests. For external requests, the client error 404 (Not Found) is returned. Internal requests are the following: Example:

### `keepalive_disable`

**Syntax**: `keepalive_disable none | browser ...;`  
**Default**: `keepalive_disable msie6;`  
**Context**: `http , server , location`  

Disables keep-alive connections with misbehaving browsers. The browser parameters specify which browsers will be affected. The value msie6 disables keep-alive connections with old versions of MSIE, once a POST request is received. The value safari disables keep-alive connections with Safari and Safari-like browsers on macOS and macOS-like operating systems. The value none enables keep-alive connections with all browsers.

### `keepalive_timeout`

**Syntax**: `keepalive_timeout timeout [ header_timeout ];`  
**Default**: `keepalive_timeout 75s;`  
**Context**: `http , server , location`  

The first parameter sets a timeout during which a keep-alive client connection will stay open on the server side. The zero value disables keep-alive client connections. The optional second parameter sets a value in the “Keep-Alive: timeout= time ” response header field. Two parameters may differ. The “Keep-Alive: timeout= time ” header field is recognized by Mozilla and Konqueror. MSIE closes keep-alive connections by itself in about 60 seconds.

### `large_client_header_buffers`

**Syntax**: `large_client_header_buffers number size ;`  
**Default**: `large_client_header_buffers 4 8k;`  
**Context**: `http , server`  

Sets the maximum number and size of buffers used for reading large client request header. A request line cannot exceed the size of one buffer, or the 414 (Request-URI Too Large) error is returned to the client. A request header field cannot exceed the size of one buffer as well, or the 400 (Bad Request) error is returned to the client. Buffers are allocated only on demand. By default, the buffer size is equal to 8K bytes. If after the end of request processing a connection is transitioned into the keep-alive state, these buffers are released. If the directive is specified on the server level, the value from the default server can be used. Details are provided in the “ Virtual server selection ” section.

### `limit_except`

**Syntax**: `limit_except method ... { ... }`  
**Default**: `—`  
**Context**: `location`  

Limits allowed HTTP methods inside a location. The method parameter can be one of the following: GET , HEAD , POST , PUT , DELETE , MKCOL , COPY , MOVE , OPTIONS , PROPFIND , PROPPATCH , LOCK , UNLOCK , or PATCH . Allowing the GET method makes the HEAD method also allowed. Access to other methods can be limited using the ngx_http_access_module , ngx_http_auth_basic_module , and ngx_http_auth_jwt_module (1.13.10) modules directives: limit_except GET { allow 192.168.1.0/32; deny all; } Please note that this will limit access to all methods except GET and HEAD.

### `limit_rate`

**Syntax**: `limit_rate rate ;`  
**Default**: `limit_rate 0;`  
**Context**: `http , server , location , if in location`  

Limits the rate of response transmission to a client. The rate is specified in bytes per second. The zero value disables rate limiting. The limit is set per a request, and so if a client simultaneously opens two connections, the overall rate will be twice as much as the specified limit. Parameter value can contain variables (1.17.0). It may be useful in cases where rate should be limited depending on a certain condition: map $slow $rate { 1 4k; 2 8k; } limit_rate $rate;

### `lingering_time`

**Syntax**: `lingering_time time ;`  
**Default**: `lingering_time 30s;`  
**Context**: `http , server , location`  

When lingering_close is in effect, this directive specifies the maximum time during which nginx will process (read and ignore) additional data coming from a client. After that, the connection will be closed, even if there will be more data.

### `lingering_timeout`

**Syntax**: `lingering_timeout time ;`  
**Default**: `lingering_timeout 5s;`  
**Context**: `http , server , location`  

When lingering_close is in effect, this directive specifies the maximum waiting time for more client data to arrive. If data are not received during this time, the connection is closed. Otherwise, the data are read and ignored, and nginx starts waiting for more data again. The “wait-read-ignore” cycle is repeated, but no longer than specified by the lingering_time directive.

### `listen`

**Syntax**: `listen address [: port ] [ default_server ] [ ssl ] [ http2 | quic ] [ proxy_protocol ] [ setfib = number ] [ fastopen = number ] [ backlog = number ] [ rcvbuf = size ] [ sndbuf = size ] [ accept_filter = filter ] [ deferred ] [ bind ] [ ipv6only = on | off ] [ reuseport ] [ multipath ] [ so_keepalive = on | off |[ keepidle ]:[ keepintvl ]:[ keepcnt ]]; listen port [ default_server ] [ ssl ] [ http2 | quic ] [ proxy_protocol ] [ setfib = number ] [ fastopen = number ] [ backlog = number ] [ rcvbuf = size ] [ sndbuf = size ] [ accept_filter = filter ] [ deferred ] [ bind ] [ ipv6only = on | off ] [ reuseport ] [ multipath ] [ so_keepalive = on | off |[ keepidle ]:[ keepintvl ]:[ keepcnt ]]; listen unix: path [ default_server ] [ ssl ] [ http2 | quic ] [ proxy_protocol ] [ backlog = number ] [ rcvbuf = size ] [ sndbuf = size ] [ accept_filter = filter ] [ deferred ] [ bind ] [ so_keepalive = on | off |[ keepidle ]:[ keepintvl ]:[ keepcnt ]];`  
**Default**: `listen *:80 | *:8000;`  
**Context**: `server`  

Sets the address and port for IP, or the path for a UNIX-domain socket on which the server will accept requests. Both address and port , or only address or only port can be specified. An address may also be a hostname, for example: listen 127.0.0.1:8000; listen 127.0.0.1; listen 8000; listen *:8000; listen localhost:8000; IPv6 addresses (0.7.36) are specified in square brackets: listen [::]:8000; listen [::1]; UNIX-domain sockets (0.8.21) are specified with the “ unix: ” prefix:

### `location`

**Syntax**: `location [ = | ~ | ~* | ^~ ] uri { ... } location @ name { ... }`  
**Default**: `—`  
**Context**: `server , location`  

Sets configuration depending on a request URI. The matching is performed against a normalized URI, after decoding the text encoded in the “ %XX ” form, resolving references to relative path components “ . ” and “ .. ”, and possible compression of two or more adjacent slashes into a single slash. A location can either be defined by a prefix string, or by a regular expression. Regular expressions are specified with the preceding “ ~* ” modifier (for case-insensitive matching), or the “ ~ ” modifier (for case-sensitive matching). To find location matching a given request, nginx first checks locations defined using the prefix strings (prefix locations). Among them, the location with the longest matching prefix is selected and remembered. Then regular expressions are checked, in the order of their appearance in the configuration file. The search of regular expressions terminates on the first match, and the corresponding configuration is used. If no match with a regular expression is found then the configuration of the prefix location remembered earlier is used.

### `log_not_found`

**Syntax**: `log_not_found on | off ;`  
**Default**: `log_not_found on;`  
**Context**: `http , server , location`  

Enables or disables logging of errors about not found files into error_log .

### `log_subrequest`

**Syntax**: `log_subrequest on | off ;`  
**Default**: `log_subrequest off;`  
**Context**: `http , server , location`  

Enables or disables logging of subrequests into access_log .

### `merge_slashes`

**Syntax**: `merge_slashes on | off ;`  
**Default**: `merge_slashes on;`  
**Context**: `http , server`  

Enables or disables compression of two or more adjacent slashes in a URI into a single slash. Note that compression is essential for the correct matching of prefix string and regular expression locations. Without it, the “ //scripts/one.php ” request would not match location /scripts/ { ... } and might be processed as a static file. So it gets converted to “ /scripts/one.php ”.

### `msie_padding`

**Syntax**: `msie_padding on | off ;`  
**Default**: `msie_padding on;`  
**Context**: `http , server , location`  

Enables or disables adding comments to responses for MSIE clients with status greater than 400 to increase the response size to 512 bytes.

### `msie_refresh`

**Syntax**: `msie_refresh on | off ;`  
**Default**: `msie_refresh off;`  
**Context**: `http , server , location`  

Enables or disables issuing refreshes instead of redirects for MSIE clients.

### `open_file_cache`

**Syntax**: `open_file_cache off ; open_file_cache max = N [ inactive = time ];`  
**Default**: `open_file_cache off;`  
**Context**: `http , server , location`  

Configures a cache that can store: The directive has the following parameters:

### `open_file_cache_errors`

**Syntax**: `open_file_cache_errors on | off ;`  
**Default**: `open_file_cache_errors off;`  
**Context**: `http , server , location`  

Enables or disables caching of file lookup errors by open_file_cache .

### `open_file_cache_min_uses`

**Syntax**: `open_file_cache_min_uses number ;`  
**Default**: `open_file_cache_min_uses 1;`  
**Context**: `http , server , location`  

Sets the minimum number of file accesses during the period configured by the inactive parameter of the open_file_cache directive, required for a file descriptor to remain open in the cache.

### `open_file_cache_valid`

**Syntax**: `open_file_cache_valid time ;`  
**Default**: `open_file_cache_valid 60s;`  
**Context**: `http , server , location`  

Sets a time after which open_file_cache elements should be validated.

### `output_buffers`

**Syntax**: `output_buffers number size ;`  
**Default**: `output_buffers 2 32k;`  
**Context**: `http , server , location`  

Sets the number and size of the buffers used for reading a response from a disk.

### `port_in_redirect`

**Syntax**: `port_in_redirect on | off ;`  
**Default**: `port_in_redirect on;`  
**Context**: `http , server , location`  

Enables or disables specifying the port in absolute redirects issued by nginx. The use of the primary server name in redirects is controlled by the server_name_in_redirect directive.

### `postpone_output`

**Syntax**: `postpone_output size ;`  
**Default**: `postpone_output 1460;`  
**Context**: `http , server , location`  

If possible, the transmission of client data will be postponed until nginx has at least size bytes of data to send. The zero value disables postponing data transmission.

### `read_ahead`

**Syntax**: `read_ahead size ;`  
**Default**: `read_ahead 0;`  
**Context**: `http , server , location`  

Sets the amount of pre-reading for the kernel when working with file. On Linux, the posix_fadvise(0, 0, 0, POSIX_FADV_SEQUENTIAL) system call is used, and so the size parameter is ignored. On FreeBSD, the fcntl(O_READAHEAD, size ) system call, supported since FreeBSD 9.0-CURRENT, is used. FreeBSD 7 has to be patched .

### `recursive_error_pages`

**Syntax**: `recursive_error_pages on | off ;`  
**Default**: `recursive_error_pages off;`  
**Context**: `http , server , location`  

Enables or disables doing several redirects using the error_page directive. The number of such redirects is limited .

### `request_pool_size`

**Syntax**: `request_pool_size size ;`  
**Default**: `request_pool_size 4k;`  
**Context**: `http , server`  

Allows accurate tuning of per-request memory allocations. This directive has minimal impact on performance and should not generally be used.

### `reset_timedout_connection`

**Syntax**: `reset_timedout_connection on | off ;`  
**Default**: `reset_timedout_connection off;`  
**Context**: `http , server , location`  

Enables or disables resetting timed out connections and connections closed with the non-standard code 444 (1.15.2). The reset is performed as follows. Before closing a socket, the SO_LINGER option is set on it with a timeout value of 0. When the socket is closed, TCP RST is sent to the client, and all memory occupied by this socket is released. This helps avoid keeping an already closed socket with filled buffers in a FIN_WAIT1 state for a long time. It should be noted that timed out keep-alive connections are closed normally.

### `resolver`

**Syntax**: `resolver address ... [ valid = time ] [ ipv4 = on | off ] [ ipv6 = on | off ] [ status_zone = zone ];`  
**Default**: `—`  
**Context**: `http , server , location`  

Configures name servers used to resolve names of upstream servers into addresses, for example: resolver 127.0.0.1 [::1]:5353; The address can be specified as a domain name or IP address, with an optional port (1.3.1, 1.2.2). If port is not specified, the port 53 is used. Name servers are queried in a round-robin fashion.

### `resolver_timeout`

**Syntax**: `resolver_timeout time ;`  
**Default**: `resolver_timeout 30s;`  
**Context**: `http , server , location`  

Sets a timeout for name resolution, for example: resolver_timeout 5s;

### `root`

**Syntax**: `root path ;`  
**Default**: `root html;`  
**Context**: `http , server , location , if in location`  

Sets the root directory for requests. For example, with the following configuration location /i/ { root /data/w3; } The /data/w3/i/top.gif file will be sent in response to the “ /i/top.gif ” request. The path value can contain variables, except $document_root and $realpath_root .

### `satisfy`

**Syntax**: `satisfy all | any ;`  
**Default**: `satisfy all;`  
**Context**: `http , server , location`  

Allows access if all ( all ) or at least one ( any ) of the ngx_http_access_module , ngx_http_auth_basic_module , ngx_http_auth_request_module , ngx_http_auth_jwt_module (1.13.10), or ngx_http_auth_oidc_module (1.27.4) modules allow access. Example: location / { satisfy any; allow 192.168.1.0/32; deny all; auth_basic "closed site"; auth_basic_user_file conf/htpasswd; }

### `send_lowat`

**Syntax**: `send_lowat size ;`  
**Default**: `send_lowat 0;`  
**Context**: `http , server , location`  

If the directive is set to a non-zero value, nginx will try to minimize the number of send operations on client sockets by using either NOTE_LOWAT flag of the kqueue method or the SO_SNDLOWAT socket option. In both cases the specified size is used. This directive is ignored on Linux, Solaris, and Windows.

### `send_timeout`

**Syntax**: `send_timeout time ;`  
**Default**: `send_timeout 60s;`  
**Context**: `http , server , location`  

Sets a timeout for transmitting a response to the client. The timeout is set only between two successive write operations, not for the transmission of the whole response. If the client does not receive anything within this time, the connection is closed.

### `sendfile`

**Syntax**: `sendfile on | off ;`  
**Default**: `sendfile off;`  
**Context**: `http , server , location , if in location`  

Enables or disables the use of sendfile() . Starting from nginx 0.8.12 and FreeBSD 5.2.1, aio can be used to pre-load data for sendfile() : location /video/ { sendfile on; tcp_nopush on; aio on; } In this configuration, sendfile() is called with the SF_NODISKIO flag which causes it not to block on disk I/O, but, instead, report back that the data are not in memory. nginx then initiates an asynchronous data load by reading one byte. On the first read, the FreeBSD kernel loads the first 128K bytes of a file into memory, although next reads will only load data in 16K chunks. This can be changed using the read_ahead directive.

### `sendfile_max_chunk`

**Syntax**: `sendfile_max_chunk size ;`  
**Default**: `sendfile_max_chunk 2m;`  
**Context**: `http , server , location`  

Limits the amount of data that can be transferred in a single sendfile() call. Without the limit, one fast connection may seize the worker process entirely.

### `server`

**Syntax**: `server { ... }`  
**Default**: `—`  
**Context**: `http`  

Sets configuration for a virtual server. There is no clear separation between IP-based (based on the IP address) and name-based (based on the “Host” request header field) virtual servers. Instead, the listen directives describe all addresses and ports that should accept connections for the server, and the server_name directive lists all server names. Example configurations are provided in the “ How nginx processes a request ” document.

### `server_name`

**Syntax**: `server_name name ...;`  
**Default**: `server_name "";`  
**Context**: `server`  

Sets names of a virtual server, for example: server { server_name example.com www.example.com; } The first name becomes the primary server name.

### `server_name_in_redirect`

**Syntax**: `server_name_in_redirect on | off ;`  
**Default**: `server_name_in_redirect off;`  
**Context**: `http , server , location`  

Enables or disables the use of the primary server name, specified by the server_name directive, in absolute redirects issued by nginx. When the use of the primary server name is disabled, the name from the “Host” request header field is used. If this field is not present, the IP address of the server is used. The use of a port in redirects is controlled by the port_in_redirect directive.

### `server_names_hash_bucket_size`

**Syntax**: `server_names_hash_bucket_size size ;`  
**Default**: `server_names_hash_bucket_size 32|64|128;`  
**Context**: `http`  

Sets the bucket size for the server names hash tables. The default value depends on the size of the processor’s cache line. The details of setting up hash tables are provided in a separate document .

### `server_names_hash_max_size`

**Syntax**: `server_names_hash_max_size size ;`  
**Default**: `server_names_hash_max_size 512;`  
**Context**: `http`  

Sets the maximum size of the server names hash tables. The details of setting up hash tables are provided in a separate document .

### `server_tokens`

**Syntax**: `server_tokens on | off | build | string ;`  
**Default**: `server_tokens on;`  
**Context**: `http , server , location`  

Enables or disables emitting nginx version on error pages and in the “Server” response header field.

### `tcp_nodelay`

**Syntax**: `tcp_nodelay on | off ;`  
**Default**: `tcp_nodelay on;`  
**Context**: `http , server , location`  

Enables or disables the use of the TCP_NODELAY option. The option is enabled when a connection is transitioned into the keep-alive state. Additionally, it is enabled on SSL connections, for unbuffered proxying, and for WebSocket proxying.

### `tcp_nopush`

**Syntax**: `tcp_nopush on | off ;`  
**Default**: `tcp_nopush off;`  
**Context**: `http , server , location`  

Enables or disables the use of the TCP_NOPUSH socket option on FreeBSD or the TCP_CORK socket option on Linux. The options are enabled only when sendfile is used. Enabling the option allows

### `try_files`

**Syntax**: `try_files file ... uri ; try_files file ... = code ;`  
**Default**: `—`  
**Context**: `server , location`  

Checks the existence of files in the specified order and uses the first found file for request processing; the processing is performed in the current context. The path to a file is constructed from the file parameter according to the root and alias directives. It is possible to check directory’s existence by specifying a slash at the end of a name, e.g. “ $uri/ ”. If none of the files were found, an internal redirect to the uri specified in the last parameter is made. For example: location /images/ { try_files $uri /images/default.gif; } location = /images/default.gif { expires 30s; } The last parameter can also point to a named location, as shown in examples below. Starting from version 0.7.51, the last parameter can also be a code : location / { try_files $uri $uri/index.html $uri.html =404; }

### `types`

**Syntax**: `types { ... }`  
**Default**: `types { text/html html; image/gif gif; image/jpeg jpg; }`  
**Context**: `http , server , location`  

Maps file name extensions to MIME types of responses. Extensions are case-insensitive. Several extensions can be mapped to one type, for example: types { application/octet-stream bin exe dll; application/octet-stream deb; application/octet-stream dmg; } A sufficiently full mapping table is distributed with nginx in the conf/mime.types file.

### `types_hash_bucket_size`

**Syntax**: `types_hash_bucket_size size ;`  
**Default**: `types_hash_bucket_size 64;`  
**Context**: `http , server , location`  

Sets the bucket size for the types hash tables. The details of setting up hash tables are provided in a separate document .

### `types_hash_max_size`

**Syntax**: `types_hash_max_size size ;`  
**Default**: `types_hash_max_size 1024;`  
**Context**: `http , server , location`  

Sets the maximum size of the types hash tables. The details of setting up hash tables are provided in a separate document .

### `underscores_in_headers`

**Syntax**: `underscores_in_headers on | off ;`  
**Default**: `underscores_in_headers off;`  
**Context**: `http , server`  

Enables or disables the use of underscores in client request header fields. When the use of underscores is disabled, request header fields whose names contain underscores are marked as invalid and become subject to the ignore_invalid_headers directive. If the directive is specified on the server level, the value from the default server can be used. Details are provided in the “ Virtual server selection ” section.

### `variables_hash_bucket_size`

**Syntax**: `variables_hash_bucket_size size ;`  
**Default**: `variables_hash_bucket_size 64;`  
**Context**: `http`  

Sets the bucket size for the variables hash table. The details of setting up hash tables are provided in a separate document .

### `variables_hash_max_size`

**Syntax**: `variables_hash_max_size size ;`  
**Default**: `variables_hash_max_size 1024;`  
**Context**: `http`  

Sets the maximum size of the variables hash table. The details of setting up hash tables are provided in a separate document .
