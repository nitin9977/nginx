# Module ngx_http_geoip_module

**Source**: https://nginx.org/en/docs/http/ngx_http_geoip_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives geoip_country geoip_city geoip_org geoip_proxy geoip_proxy_recursive The ngx_http_geoip_module module (0.8.6+) creates variables with values depending on the client IP address, using the precompiled MaxMind databases. When using the databases with IPv6 support (1.3.12, 1.2.7), IPv4 addresses are looked up as IPv4-mapped IPv6 addresses. This module is not built by default, it should be enabled with the --with-http_geoip_module configuration parameter. This module requires the MaxMind GeoIP library.

## Example Configuration

```nginx
http {
    geoip_country         GeoIP.dat;
    geoip_city            GeoLiteCity.dat;
    geoip_proxy           192.168.100.0/24;
    geoip_proxy           2001:0db8::/32;
    geoip_proxy_recursive on;
    ...
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `geoip_country` | geoip_country file ; | — | http |
| `geoip_city` | geoip_city file ; | — | http |

## Directive Details

### `geoip_country`

**Syntax**: `geoip_country file ;`  
**Default**: `—`  
**Context**: `http`  

Specifies a database used to determine the country depending on the client IP address. The following variables are available when using this database:

### `geoip_city`

**Syntax**: `geoip_city file ;`  
**Default**: `—`  
**Context**: `http`  

Specifies a database used to determine the country, region, and city depending on the client IP address. The following variables are available when using this database:
