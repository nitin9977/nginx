# Module ngx_http_autoindex_module

**Source**: https://nginx.org/en/docs/http/ngx_http_autoindex_module.html  

## Overview

Example Configuration Directives autoindex autoindex_exact_size autoindex_format autoindex_localtime The ngx_http_autoindex_module module processes requests ending with the slash character (‘ / ’) and produces a directory listing. Usually a request is passed to the ngx_http_autoindex_module module when the ngx_http_index_module module cannot find an index file.

## Example Configuration

```nginx
location / {
    autoindex on;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `autoindex` | autoindex on \| off ; | autoindex off; | http , server , location |
| `autoindex_exact_size` | autoindex_exact_size on \| off ; | autoindex_exact_size on; | http , server , location |
| `autoindex_localtime` | autoindex_localtime on \| off ; | autoindex_localtime off; | http , server , location |

## Directive Details

### `autoindex`

**Syntax**: `autoindex on | off ;`  
**Default**: `autoindex off;`  
**Context**: `http , server , location`  

Enables or disables the directory listing output.

### `autoindex_exact_size`

**Syntax**: `autoindex_exact_size on | off ;`  
**Default**: `autoindex_exact_size on;`  
**Context**: `http , server , location`  

For the HTML format , specifies whether exact file sizes should be output in the directory listing, or rather rounded to kilobytes, megabytes, and gigabytes.

### `autoindex_localtime`

**Syntax**: `autoindex_localtime on | off ;`  
**Default**: `autoindex_localtime off;`  
**Context**: `http , server , location`  

For the HTML format , specifies whether times in the directory listing should be output in the local time zone or UTC.
