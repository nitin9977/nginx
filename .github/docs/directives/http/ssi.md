# Module ngx_http_ssi_module

**Source**: https://nginx.org/en/docs/http/ngx_http_ssi_module.html  

## Overview

Example Configuration Directives ssi ssi_last_modified ssi_min_file_chunk ssi_silent_errors ssi_types ssi_value_length SSI Commands Embedded Variables The ngx_http_ssi_module module is a filter that processes SSI (Server Side Includes) commands in responses passing through it. Currently, the list of supported SSI commands is incomplete.

## Example Configuration

```nginx
location / {
    ssi on;
    ...
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `ssi` | ssi on \| off ; | ssi off; | http , server , location , if in location |
| `ssi_min_file_chunk` | ssi_min_file_chunk size ; | ssi_min_file_chunk 1k; | http , server , location |
| `ssi_silent_errors` | ssi_silent_errors on \| off ; | ssi_silent_errors off; | http , server , location |
| `ssi_types` | ssi_types mime-type ...; | ssi_types text/html; | http , server , location |
| `ssi_value_length` | ssi_value_length length ; | ssi_value_length 256; | http , server , location |

## Directive Details

### `ssi`

**Syntax**: `ssi on | off ;`  
**Default**: `ssi off;`  
**Context**: `http , server , location , if in location`  

Enables or disables processing of SSI commands in responses.

### `ssi_min_file_chunk`

**Syntax**: `ssi_min_file_chunk size ;`  
**Default**: `ssi_min_file_chunk 1k;`  
**Context**: `http , server , location`  

Sets the minimum size for parts of a response stored on disk, starting from which it makes sense to send them using sendfile .

### `ssi_silent_errors`

**Syntax**: `ssi_silent_errors on | off ;`  
**Default**: `ssi_silent_errors off;`  
**Context**: `http , server , location`  

If enabled, suppresses the output of the “ [an error occurred while processing the directive] ” string if an error occurred during SSI processing.

### `ssi_types`

**Syntax**: `ssi_types mime-type ...;`  
**Default**: `ssi_types text/html;`  
**Context**: `http , server , location`  

Enables processing of SSI commands in responses with the specified MIME types in addition to “ text/html ”. The special value “ * ” matches any MIME type (0.8.29).

### `ssi_value_length`

**Syntax**: `ssi_value_length length ;`  
**Default**: `ssi_value_length 256;`  
**Context**: `http , server , location`  

Sets the maximum length of parameter values in SSI commands.
