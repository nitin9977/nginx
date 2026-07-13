# Module ngx_http_log_module

**Source**: https://nginx.org/en/docs/http/ngx_http_log_module.html  

## Overview

Example Configuration Directives access_log log_format open_log_file_cache The ngx_http_log_module module writes request logs in the specified format. Requests are logged in the context of a location where processing ends. It may be different from the original location, if an internal redirect happens during request processing.

## Example Configuration

```nginx
log_format compression '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $bytes_sent '
                       '"$http_referer" "$http_user_agent" "$gzip_ratio"';

access_log /spool/logs/nginx-access.log compression buffer=32k;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `access_log` | access_log path [ format [ buffer = size ] [ gzip[= level ] ] [ flush = time ] [ if = condition ]]; access_log off ; | access_log logs/access.log combined; | http , server , location , if in location , limit_except |
| `log_format` | log_format name [ escape = default \| json \| none ] string ...; | log_format combined "..."; | http |
| `open_log_file_cache` | open_log_file_cache max = N [ inactive = time ] [ min_uses = N ] [ valid = time ]; open_log_file_cache off ; | open_log_file_cache off; | http , server , location |

## Directive Details

### `access_log`

**Syntax**: `access_log path [ format [ buffer = size ] [ gzip[= level ] ] [ flush = time ] [ if = condition ]]; access_log off ;`  
**Default**: `access_log logs/access.log combined;`  
**Context**: `http , server , location , if in location , limit_except`  

Sets the path, format, and configuration for a buffered log write. Several logs can be specified on the same configuration level. Logging to syslog can be configured by specifying the “ syslog: ” prefix in the first parameter. The special value off cancels all access_log directives on the current level. If the format is not specified then the predefined “ combined ” format is used. If either the buffer or gzip (1.3.10, 1.2.7) parameter is used, writes to log will be buffered.

### `log_format`

**Syntax**: `log_format name [ escape = default | json | none ] string ...;`  
**Default**: `log_format combined "...";`  
**Context**: `http`  

Specifies log format.

### `open_log_file_cache`

**Syntax**: `open_log_file_cache max = N [ inactive = time ] [ min_uses = N ] [ valid = time ]; open_log_file_cache off ;`  
**Default**: `open_log_file_cache off;`  
**Context**: `http , server , location`  

Defines a cache that stores the file descriptors of frequently used logs whose names contain variables. The directive has the following parameters: Usage example:
