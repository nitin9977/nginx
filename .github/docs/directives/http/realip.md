# Module ngx_http_realip_module

**Source**: https://nginx.org/en/docs/http/ngx_http_realip_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives set_real_ip_from real_ip_header real_ip_recursive Embedded Variables The ngx_http_realip_module module is used to change the client address and optional port to those sent in the specified header field. This module is not built by default, it should be enabled with the --with-http_realip_module configuration parameter.

## Example Configuration

```nginx
set_real_ip_from  192.168.1.0/24;
set_real_ip_from  192.168.2.1;
set_real_ip_from  2001:0db8::/32;
real_ip_header    X-Forwarded-For;
real_ip_recursive on;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `set_real_ip_from` | set_real_ip_from address \| CIDR \| unix: ; | — | http , server , location |
| `real_ip_header` | real_ip_header field \| X-Real-IP \| X-Forwarded-For \| proxy_protocol ; | real_ip_header X-Real-IP; | http , server , location |

## Directive Details

### `set_real_ip_from`

**Syntax**: `set_real_ip_from address | CIDR | unix: ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines trusted addresses that are known to send correct replacement addresses. If the special value unix: is specified, all UNIX-domain sockets will be trusted. Trusted addresses may also be specified using a hostname (1.13.1).

### `real_ip_header`

**Syntax**: `real_ip_header field | X-Real-IP | X-Forwarded-For | proxy_protocol ;`  
**Default**: `real_ip_header X-Real-IP;`  
**Context**: `http , server , location`  

Defines the request header field whose value will be used to replace the client address. The request header field value that contains an optional port is also used to replace the client port (1.11.0). The address and port should be specified according to RFC 3986 . The proxy_protocol parameter (1.5.12) changes the client address to the one from the PROXY protocol header. The PROXY protocol must be previously enabled by setting the proxy_protocol parameter in the listen directive.
