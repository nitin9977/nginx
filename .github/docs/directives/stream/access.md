# Module ngx_stream_access_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_access_module.html  

## Overview

Example Configuration Directives allow deny The ngx_stream_access_module module (1.9.2) allows limiting access to certain client addresses.

## Example Configuration

```nginx
server {
    ...
    deny  192.168.1.1;
    allow 192.168.1.0/24;
    allow 10.1.1.0/16;
    allow 2001:0db8::/32;
    deny  all;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `allow` | allow address \| CIDR \| unix: \| all ; | — | stream , server |
| `deny` | deny address \| CIDR \| unix: \| all ; | — | stream , server |

## Directive Details

### `allow`

**Syntax**: `allow address | CIDR | unix: | all ;`  
**Default**: `—`  
**Context**: `stream , server`  

Allows access for the specified network or address. If the special value unix: is specified, allows access for all UNIX-domain sockets.

### `deny`

**Syntax**: `deny address | CIDR | unix: | all ;`  
**Default**: `—`  
**Context**: `stream , server`  

Denies access for the specified network or address. If the special value unix: is specified, denies access for all UNIX-domain sockets.
