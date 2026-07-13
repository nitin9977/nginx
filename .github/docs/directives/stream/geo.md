# Module ngx_stream_geo_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_geo_module.html  

## Overview

Example Configuration Directives geo The ngx_stream_geo_module module (1.11.3) creates variables with values depending on the client IP address.

## Example Configuration

```nginx
geo $geo {
    default        0;

    127.0.0.1      2;
    192.168.1.0/24 1;
    10.1.0.0/16    1;

    ::1            2;
    2001:0db8::/32 1;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `geo` | geo [ $address ] $variable { ... } | — | stream |

## Directive Details

### `geo`

**Syntax**: `geo [ $address ] $variable { ... }`  
**Default**: `—`  
**Context**: `stream`  

Describes the dependency of values of the specified variable on the client IP address. By default, the address is taken from the $remote_addr variable, but it can also be taken from another variable, for example: geo $arg_remote_addr $geo { ...; }
