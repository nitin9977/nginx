# Module ngx_http_gzip_module

**Source**: https://nginx.org/en/docs/http/ngx_http_gzip_module.html  

## Overview

Example Configuration Directives gzip gzip_buffers gzip_comp_level gzip_disable gzip_http_version gzip_min_length gzip_proxied gzip_types gzip_vary Embedded Variables The ngx_http_gzip_module module is a filter that compresses responses using the “gzip” method. This often helps to reduce the size of transmitted data by half or even more. When using the SSL/TLS protocol, compressed responses may be subject to BREACH attacks.

## Example Configuration

```nginx
gzip            on;
gzip_min_length 1000;
gzip_proxied    expired no-cache no-store private auth;
gzip_types      text/plain application/xml;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `gzip` | gzip on \| off ; | gzip off; | http , server , location , if in location |
| `gzip_buffers` | gzip_buffers number size ; | gzip_buffers 32 4k\|16 8k; | http , server , location |
| `gzip_comp_level` | gzip_comp_level level ; | gzip_comp_level 1; | http , server , location |
| `gzip_http_version` | gzip_http_version 1.0 \| 1.1 ; | gzip_http_version 1.1; | http , server , location |
| `gzip_min_length` | gzip_min_length length ; | gzip_min_length 20; | http , server , location |
| `gzip_proxied` | gzip_proxied off \| expired \| no-cache \| no-store \| private \| no_last_modified \| no_etag \| auth \| any ...; | gzip_proxied off; | http , server , location |
| `gzip_types` | gzip_types mime-type ...; | gzip_types text/html; | http , server , location |
| `gzip_vary` | gzip_vary on \| off ; | gzip_vary off; | http , server , location |

## Directive Details

### `gzip`

**Syntax**: `gzip on | off ;`  
**Default**: `gzip off;`  
**Context**: `http , server , location , if in location`  

Enables or disables gzipping of responses.

### `gzip_buffers`

**Syntax**: `gzip_buffers number size ;`  
**Default**: `gzip_buffers 32 4k|16 8k;`  
**Context**: `http , server , location`  

Sets the number and size of buffers used to compress a response. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.

### `gzip_comp_level`

**Syntax**: `gzip_comp_level level ;`  
**Default**: `gzip_comp_level 1;`  
**Context**: `http , server , location`  

Sets a gzip compression level of a response. Acceptable values are in the range from 1 to 9.

### `gzip_http_version`

**Syntax**: `gzip_http_version 1.0 | 1.1 ;`  
**Default**: `gzip_http_version 1.1;`  
**Context**: `http , server , location`  

Sets the minimum HTTP version of a request required to compress a response.

### `gzip_min_length`

**Syntax**: `gzip_min_length length ;`  
**Default**: `gzip_min_length 20;`  
**Context**: `http , server , location`  

Sets the minimum length of a response that will be gzipped. The length is determined only from the “Content-Length” response header field.

### `gzip_proxied`

**Syntax**: `gzip_proxied off | expired | no-cache | no-store | private | no_last_modified | no_etag | auth | any ...;`  
**Default**: `gzip_proxied off;`  
**Context**: `http , server , location`  

Enables or disables gzipping of responses for proxied requests depending on the request and response. The fact that the request is proxied is determined by the presence of the “Via” request header field. The directive accepts multiple parameters:

### `gzip_types`

**Syntax**: `gzip_types mime-type ...;`  
**Default**: `gzip_types text/html;`  
**Context**: `http , server , location`  

Enables gzipping of responses for the specified MIME types in addition to “ text/html ”. The special value “ * ” matches any MIME type (0.8.29). Responses with the “ text/html ” type are always compressed.

### `gzip_vary`

**Syntax**: `gzip_vary on | off ;`  
**Default**: `gzip_vary off;`  
**Context**: `http , server , location`  

Enables or disables inserting the “Vary: Accept-Encoding” response header field if the directives gzip , gzip_static , or gunzip are active.
