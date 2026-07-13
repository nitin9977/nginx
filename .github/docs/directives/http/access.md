# Module ngx_http_access_module

**Source**: https://nginx.org/en/docs/http/ngx_http_access_module.html  

## Overview

Example Configuration Directives allow deny The ngx_http_access_module module allows limiting access to certain client addresses. Access can also be limited by password , by the result of subrequest , or by JWT . Simultaneous limitation of access by address and by password is controlled by the satisfy directive.

## Example Configuration

```nginx
location / {
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
| `allow` | allow address \| CIDR \| unix: \| all ; | — | http , server , location , limit_except |
| `deny` | deny address \| CIDR \| unix: \| all ; | — | http , server , location , limit_except |

## Directive Details

### `allow`

**Syntax**: `allow address | CIDR | unix: | all ;`  
**Default**: `—`  
**Context**: `http , server , location , limit_except`  

Allows access for the specified network or address. If the special value unix: is specified (1.5.1), allows access for all UNIX-domain sockets. Several allow directives can be specified on the same level. These directives are inherited from the previous configuration level if and only if there are no allow and deny directives defined on the current level.

### `deny`

**Syntax**: `deny address | CIDR | unix: | all ;`  
**Default**: `—`  
**Context**: `http , server , location , limit_except`  

Denies access for the specified network or address. If the special value unix: is specified (1.5.1), denies access for all UNIX-domain sockets. Several deny directives can be specified on the same level. These directives are inherited from the previous configuration level if and only if there are no allow and deny directives defined on the current level.
