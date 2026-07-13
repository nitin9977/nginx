# Module ngx_http_map_module

**Source**: https://nginx.org/en/docs/http/ngx_http_map_module.html  

## Overview

Example Configuration Directives map map_hash_bucket_size map_hash_max_size The ngx_http_map_module module creates variables whose values depend on values of other variables.

## Example Configuration

```nginx
map $http_host $name {
    hostnames;

    default       0;

    example.com   1;
    *.example.com 1;
    example.org   2;
    *.example.org 2;
    .example.net  3;
    wap.*         4;
}

map $http_user_agent $mobile {
    default       0;
    "~Opera Mini" 1;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `map` | map string $variable { ... } | — | http |
| `map_hash_bucket_size` | map_hash_bucket_size size ; | map_hash_bucket_size 32\|64\|128; | http |
| `map_hash_max_size` | map_hash_max_size size ; | map_hash_max_size 2048; | http |

## Directive Details

### `map`

**Syntax**: `map string $variable { ... }`  
**Default**: `—`  
**Context**: `http`  

Creates a new variable whose value depends on values of one or more of the source variables specified in the first parameter.

### `map_hash_bucket_size`

**Syntax**: `map_hash_bucket_size size ;`  
**Default**: `map_hash_bucket_size 32|64|128;`  
**Context**: `http`  

Sets the bucket size for the map variables hash tables. Default value depends on the processor’s cache line size. The details of setting up hash tables are provided in a separate document .

### `map_hash_max_size`

**Syntax**: `map_hash_max_size size ;`  
**Default**: `map_hash_max_size 2048;`  
**Context**: `http`  

Sets the maximum size of the map variables hash tables. The details of setting up hash tables are provided in a separate document .
