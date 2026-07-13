# Module ngx_http_charset_module

**Source**: https://nginx.org/en/docs/http/ngx_http_charset_module.html  

## Overview

Example Configuration Directives charset charset_map charset_types override_charset source_charset The ngx_http_charset_module module adds the specified charset to the “Content-Type” response header field. In addition, the module can convert data from one charset to another, with some limitations: conversion is performed one way — from server to client, only single-byte charsets can be converted or single-byte charsets to/from UTF-8.

## Example Configuration

```nginx
include        conf/koi-win;

charset        windows-1251;
source_charset koi8-r;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `charset` | charset charset \| off ; | charset off; | http , server , location , if in location |
| `charset_map` | charset_map charset1 charset2 { ... } | — | http |
| `override_charset` | override_charset on \| off ; | override_charset off; | http , server , location , if in location |
| `source_charset` | source_charset charset ; | — | http , server , location , if in location |

## Directive Details

### `charset`

**Syntax**: `charset charset | off ;`  
**Default**: `charset off;`  
**Context**: `http , server , location , if in location`  

Adds the specified charset to the “Content-Type” response header field. If this charset is different from the charset specified in the source_charset directive, a conversion is performed. The parameter off cancels the addition of charset to the “Content-Type” response header field. A charset can be defined with a variable:

### `charset_map`

**Syntax**: `charset_map charset1 charset2 { ... }`  
**Default**: `—`  
**Context**: `http`  

Describes the conversion table from one charset to another. A reverse conversion table is built using the same data. Character codes are given in hexadecimal. Missing characters in the range 80-FF are replaced with “ ? ”. When converting from UTF-8, characters missing in a one-byte charset are replaced with “ &#XXXX; ”. Example: charset_map koi8-r windows-1251 { C0 FE ; # small yu C1 E0 ; # small a C2 E1 ; # small b C3 F6 ; # small ts ... }

### `override_charset`

**Syntax**: `override_charset on | off ;`  
**Default**: `override_charset off;`  
**Context**: `http , server , location , if in location`  

Determines whether a conversion should be performed for answers received from a proxied or a FastCGI/uwsgi/SCGI/gRPC server when the answers already carry a charset in the “Content-Type” response header field. If conversion is enabled, a charset specified in the received response is used as a source charset.

### `source_charset`

**Syntax**: `source_charset charset ;`  
**Default**: `—`  
**Context**: `http , server , location , if in location`  

Defines the source charset of a response. If this charset is different from the charset specified in the charset directive, a conversion is performed.
