# Module ngx_http_auth_basic_module

**Source**: https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html  

## Overview

Example Configuration Directives auth_basic auth_basic_user_file The ngx_http_auth_basic_module module allows limiting access to resources by validating the user name and password using the “HTTP Basic Authentication” protocol. Access can also be limited by address , by the result of subrequest , or by JWT . Simultaneous limitation of access by address and by password is controlled by the satisfy directive.

## Example Configuration

```nginx
location / {
    auth_basic           "closed site";
    auth_basic_user_file conf/htpasswd;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `auth_basic` | auth_basic string \| off ; | auth_basic off; | http , server , location , limit_except |
| `auth_basic_user_file` | auth_basic_user_file file ; | — | http , server , location , limit_except |

## Directive Details

### `auth_basic`

**Syntax**: `auth_basic string | off ;`  
**Default**: `auth_basic off;`  
**Context**: `http , server , location , limit_except`  

Enables validation of user name and password using the “HTTP Basic Authentication” protocol. The specified parameter is used as a realm . Parameter value can contain variables (1.3.10, 1.2.7). The special value off cancels the effect of the auth_basic directive inherited from the previous configuration level.

### `auth_basic_user_file`

**Syntax**: `auth_basic_user_file file ;`  
**Default**: `—`  
**Context**: `http , server , location , limit_except`  

Specifies a file that keeps user names and passwords, in the following format: # comment name1:password1 name2:password2:comment name3:password3 The file name can contain variables. The following password types are supported:
