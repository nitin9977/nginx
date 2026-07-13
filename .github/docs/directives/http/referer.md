# Module ngx_http_referer_module

**Source**: https://nginx.org/en/docs/http/ngx_http_referer_module.html  

## Overview

Example Configuration Directives referer_hash_bucket_size referer_hash_max_size valid_referers Embedded Variables The ngx_http_referer_module module is used to block access to a site for requests with invalid values in the “Referer” header field. It should be kept in mind that fabricating a request with an appropriate “Referer” field value is quite easy, and so the intended purpose of this module is not to block such requests thoroughly but to block the mass flow of requests sent by regular browsers. It should also be taken into consideration that regular browsers may not send the “Referer” field even for valid requests.

## Example Configuration

```nginx
valid_referers none blocked server_names
               *.example.com example.* www.example.org/galleries/
               ~\.google\.;

if ($invalid_referer) {
    return 403;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `valid_referers` | valid_referers none \| blocked \| server_names \| string ...; | — | server , location |

## Directive Details

### `valid_referers`

**Syntax**: `valid_referers none | blocked | server_names | string ...;`  
**Default**: `—`  
**Context**: `server , location`  

Specifies the “Referer” request header field values that will cause the embedded $invalid_referer variable to be set to an empty string. Otherwise, the variable will be set to “ 1 ”. Search for a match is case-insensitive. Parameters can be as follows:
