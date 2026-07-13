# Module ngx_stream_realip_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_realip_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives set_real_ip_from Embedded Variables The ngx_stream_realip_module module is used to change the client address and port to the ones sent in the PROXY protocol header (1.11.4). The PROXY protocol must be previously enabled by setting the proxy_protocol parameter in the listen directive. This module is not built by default, it should be enabled with the --with-stream_realip_module configuration parameter.

## Example Configuration

```nginx
listen 12345 proxy_protocol;

set_real_ip_from  192.168.1.0/24;
set_real_ip_from  192.168.2.1;
set_real_ip_from  2001:0db8::/32;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `set_real_ip_from` | set_real_ip_from address \| CIDR \| unix: ; | — | stream , server |

## Directive Details

### `set_real_ip_from`

**Syntax**: `set_real_ip_from address | CIDR | unix: ;`  
**Default**: `—`  
**Context**: `stream , server`  

Defines trusted addresses that are known to send correct replacement addresses. If the special value unix: is specified, all UNIX-domain sockets will be trusted.
