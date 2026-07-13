# Module ngx_stream_log_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_log_module.html  

## Overview

Example Configuration Directives access_log log_format open_log_file_cache The ngx_stream_log_module module (1.11.4) writes session logs in the specified format.

## Example Configuration

```nginx
log_format basic '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time';

access_log /spool/logs/nginx-access.log basic buffer=32k;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `access_log` | access_log path format [ buffer = size ] [ gzip[= level ] ] [ flush = time ] [ if = condition ]; access_log off ; | access_log off; | stream , server |
| `log_format` | log_format name [ escape = default \| json \| none ] string ...; | — | stream |
| `open_log_file_cache` | open_log_file_cache max = N [ inactive = time ] [ min_uses = N ] [ valid = time ]; open_log_file_cache off ; | open_log_file_cache off; | stream , server |

## Directive Details

### `access_log`

**Syntax**: `access_log path format [ buffer = size ] [ gzip[= level ] ] [ flush = time ] [ if = condition ]; access_log off ;`  
**Default**: `access_log off;`  
**Context**: `stream , server`  

Sets the path, format , and configuration for a buffered log write. Several logs can be specified on the same configuration level. Logging to syslog can be configured by specifying the “ syslog: ” prefix in the first parameter. The special value off cancels all access_log directives on the current level. If either the buffer or gzip parameter is used, writes to log will be buffered.

### `log_format`

**Syntax**: `log_format name [ escape = default | json | none ] string ...;`  
**Default**: `—`  
**Context**: `stream`  

Specifies the log format, for example: log_format proxy '$remote_addr [$time_local] ' '$protocol $status $bytes_sent $bytes_received ' '$session_time "$upstream_addr" ' '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

### `open_log_file_cache`

**Syntax**: `open_log_file_cache max = N [ inactive = time ] [ min_uses = N ] [ valid = time ]; open_log_file_cache off ;`  
**Default**: `open_log_file_cache off;`  
**Context**: `stream , server`  

Defines a cache that stores the file descriptors of frequently used logs whose names contain variables. The directive has the following parameters: Usage example:
