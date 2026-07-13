# Module ngx_google_perftools_module

**Source**: https://nginx.org/en/docs/ngx_google_perftools_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives google_perftools_profiles The ngx_google_perftools_module module (0.6.29) enables profiling of nginx worker processes using Google Performance Tools . The module is intended for nginx developers. This module is not built by default, it should be enabled with the --with-google_perftools_module configuration parameter. This module requires the gperftools library.

## Example Configuration

```nginx
google_perftools_profiles /path/to/profile;
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `google_perftools_profiles` | google_perftools_profiles file ; | — | main |

## Directive Details

### `google_perftools_profiles`

**Syntax**: `google_perftools_profiles file ;`  
**Default**: `—`  
**Context**: `main`  

Sets a file name that keeps profiling information of nginx worker process. The ID of the worker process is always a part of the file name and is appended to the end of the file name, after a dot.
