# Module ngx_stream_map_module

**Source**: https://nginx.org/en/docs/stream/ngx_stream_map_module.html  

## Overview

Example Configuration Directives map map_hash_bucket_size map_hash_max_size The ngx_stream_map_module module (1.11.2) creates variables whose values depend on values of other variables.

## Example Configuration

```nginx
map $remote_addr $limit {
    127.0.0.1    "";
    default      $binary_remote_addr;
}

limit_conn_zone $limit zone=addr:10m;
limit_conn addr 1;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `map` | map string $variable { ... } | — | stream |
| `map_hash_bucket_size` | map_hash_bucket_size size ; | map_hash_bucket_size 32\|64\|128; | stream |
| `map_hash_max_size` | map_hash_max_size size ; | map_hash_max_size 2048; | stream |

## Directive Details

### `map`

**Syntax**: `map string $variable { ... }`  
**Default**: `—`  
**Context**: `stream`  

Creates a new variable whose value depends on values of one or more of the source variables specified in the first parameter.

### `map_hash_bucket_size`

**Syntax**: `map_hash_bucket_size size ;`  
**Default**: `map_hash_bucket_size 32|64|128;`  
**Context**: `stream`  

Sets the bucket size for the map variables hash tables. Default value depends on the processor’s cache line size. The details of setting up hash tables are provided in a separate document .

### `map_hash_max_size`

**Syntax**: `map_hash_max_size size ;`  
**Default**: `map_hash_max_size 2048;`  
**Context**: `stream`  

Sets the maximum size of the map variables hash tables. The details of setting up hash tables are provided in a separate document .
