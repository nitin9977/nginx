# Module ngx_stream_geoip_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_geoip_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives geoip_country geoip_city geoip_org The ngx_stream_geoip_module module (1.11.3) creates variables with values depending on the client IP address, using the precompiled MaxMind databases. When using the databases with IPv6 support, IPv4 addresses are looked up as IPv4-mapped IPv6 addresses. This module is not built by default, it should be enabled with the --with-stream_geoip_module configuration parameter. This module requires the MaxMind GeoIP library.

## Example Configuration

```nginx
stream {
    geoip_country         GeoIP.dat;
    geoip_city            GeoLiteCity.dat;

    map $geoip_city_continent_code $nearest_server {
        default        example.com;
        EU          eu.example.com;
        NA          na.example.com;
        AS          as.example.com;
    }
   ...
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `geoip_country` | geoip_country file ; | — | stream |
| `geoip_city` | geoip_city file ; | — | stream |
| `geoip_org` | geoip_org file ; | — | stream |

## Directive Details

### `geoip_country`

**Syntax**: `geoip_country file ;`  
**Default**: `—`  
**Context**: `stream`  

Specifies a database used to determine the country depending on the client IP address. The following variables are available when using this database:

### `geoip_city`

**Syntax**: `geoip_city file ;`  
**Default**: `—`  
**Context**: `stream`  

Specifies a database used to determine the country, region, and city depending on the client IP address. The following variables are available when using this database:

### `geoip_org`

**Syntax**: `geoip_org file ;`  
**Default**: `—`  
**Context**: `stream`  

Specifies a database used to determine the organization depending on the client IP address. The following variable is available when using this database:
