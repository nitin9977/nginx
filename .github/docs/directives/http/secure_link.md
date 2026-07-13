# Module ngx_http_secure_link_module

**Source**: https://nginx.org/en/docs/http/ngx_http_secure_link_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Directives secure_link secure_link_md5 secure_link_secret Embedded Variables The ngx_http_secure_link_module module (0.7.18) is used to check authenticity of requested links, protect resources from unauthorized access, and limit link lifetime. The authenticity of a requested link is verified by comparing the checksum value passed in a request with the value computed for the request. If a link has a limited lifetime and the time has expired, the link is considered outdated. The status of these checks is made available in the $secure_link variable. The module provides two alternative operation modes. The first mode is enabled by the secure_link_secret directive and is used to check authenticity of requested links as well as protect resources from unauthorized access. The second mode (0.8.50)

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `secure_link` | secure_link expression ; | — | http , server , location |
| `secure_link_md5` | secure_link_md5 expression ; | — | http , server , location |
| `secure_link_secret` | secure_link_secret word ; | — | location |

## Directive Details

### `secure_link`

**Syntax**: `secure_link expression ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines a string with variables from which the checksum value and lifetime of a link will be extracted. Variables used in an expression are usually associated with a request; see example below. The checksum value extracted from the string is compared with the MD5 hash value of the expression defined by the secure_link_md5 directive. If the checksums are different, the $secure_link variable is set to an empty string. If the checksums are the same, the link lifetime is checked. If the link has a limited lifetime and the time has expired, the $secure_link variable is set to “ 0 ”. Otherwise, it is set to “ 1 ”. The MD5 hash value passed in a request is encoded in base64url .

### `secure_link_md5`

**Syntax**: `secure_link_md5 expression ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines an expression for which the MD5 hash value will be computed and compared with the value passed in a request. The expression should contain the secured part of a link (resource) and a secret ingredient. If the link has a limited lifetime, the expression should also contain $secure_link_expires . To prevent unauthorized access, the expression may contain some information about the client, such as its address and browser version.

### `secure_link_secret`

**Syntax**: `secure_link_secret word ;`  
**Default**: `—`  
**Context**: `location`  

Defines a secret word used to check authenticity of requested links. The full URI of a requested link looks as follows: / prefix / hash / link where hash is a hexadecimal representation of the MD5 hash computed for the concatenation of the link and secret word, and prefix is an arbitrary string without slashes.
