# Module ngx_http_rewrite_module

**Source**: https://nginx.org/en/docs/http/ngx_http_rewrite_module.html  

## Overview

Directives break if return rewrite rewrite_log set uninitialized_variable_warn Internal Implementation The ngx_http_rewrite_module module is used to change request URI using PCRE regular expressions, return redirects, and conditionally select configurations. The break , if , return , rewrite , and set directives are processed in the following order: the directives of this module specified on the server level are executed sequentially; repeatedly: a location is searched based on a request URI; the directives of this module specified inside the found location are executed sequentially; the loop is repeated if a request URI was rewritten , but not more than 10 times .

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `break` | break ; | — | server , location , if |
| `if` | if ( condition ) { ... } | — | server , location |
| `return` | return code [ text ]; return code URL ; return URL ; | — | server , location , if |
| `rewrite` | rewrite regex replacement [ flag ]; | — | server , location , if |
| `rewrite_log` | rewrite_log on \| off ; | rewrite_log off; | http , server , location , if |
| `set` | set $variable value ; | — | server , location , if |
| `uninitialized_variable_warn` | uninitialized_variable_warn on \| off ; | uninitialized_variable_warn on; | http , server , location , if |

## Directive Details

### `break`

**Syntax**: `break ;`  
**Default**: `—`  
**Context**: `server , location , if`  

Stops processing the current set of ngx_http_rewrite_module directives. If a directive is specified inside the location , further processing of the request continues in this location. Example:

### `if`

**Syntax**: `if ( condition ) { ... }`  
**Default**: `—`  
**Context**: `server , location`  

The specified condition is evaluated. If true, this module directives specified inside the braces are executed, and the request is assigned the configuration inside the if directive. Configurations inside the if directives are inherited from the previous configuration level. A condition may be any of the following:

### `return`

**Syntax**: `return code [ text ]; return code URL ; return URL ;`  
**Default**: `—`  
**Context**: `server , location , if`  

Stops processing and returns the specified code to a client. The non-standard code 444 closes a connection without sending a response header. Starting from version 0.8.42, it is possible to specify either a redirect URL (for codes 301, 302, 303, 307, and 308) or the response body text (for other codes). A response body text and redirect URL can contain variables. As a special case, a redirect URL can be specified as a URI local to this server, in which case the full redirect URL is formed according to the request scheme ( $scheme ) and the server_name_in_redirect and port_in_redirect directives. In addition, a URL for temporary redirect with the code 302 can be specified as the sole parameter. Such a parameter should start with the “ http:// ”, “ https:// ”, or “ $scheme ” string. A URL can contain variables.

### `rewrite`

**Syntax**: `rewrite regex replacement [ flag ];`  
**Default**: `—`  
**Context**: `server , location , if`  

If the specified regular expression matches a request URI, URI is changed as specified in the replacement string. The rewrite directives are executed sequentially in order of their appearance in the configuration file. It is possible to terminate further processing of the directives using flags. If a replacement string starts with “ http:// ”, “ https:// ”, or “ $scheme ”, the processing stops and the redirect is returned to a client. An optional flag parameter can be one of: The full redirect URL is formed according to the request scheme ( $scheme ) and the server_name_in_redirect and port_in_redirect directives.

### `rewrite_log`

**Syntax**: `rewrite_log on | off ;`  
**Default**: `rewrite_log off;`  
**Context**: `http , server , location , if`  

Enables or disables logging of ngx_http_rewrite_module module directives processing results into the error_log at the notice level.

### `set`

**Syntax**: `set $variable value ;`  
**Default**: `—`  
**Context**: `server , location , if`  

Sets a value for the specified variable . The value can contain text, variables, and their combination.

### `uninitialized_variable_warn`

**Syntax**: `uninitialized_variable_warn on | off ;`  
**Default**: `uninitialized_variable_warn on;`  
**Context**: `http , server , location , if`  

Controls whether warnings about uninitialized variables are logged.
