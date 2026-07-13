# Module ngx_mail_proxy_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_proxy_module.html  

## Overview

Directives proxy_buffer proxy_pass_error_message proxy_protocol proxy_smtp_auth proxy_timeout xclient

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `proxy_buffer` | proxy_buffer size ; | proxy_buffer 4k\|8k; | mail , server |
| `proxy_pass_error_message` | proxy_pass_error_message on \| off ; | proxy_pass_error_message off; | mail , server |
| `proxy_timeout` | proxy_timeout timeout ; | proxy_timeout 24h; | mail , server |
| `xclient` | xclient on \| off ; | xclient on; | mail , server |

## Directive Details

### `proxy_buffer`

**Syntax**: `proxy_buffer size ;`  
**Default**: `proxy_buffer 4k|8k;`  
**Context**: `mail , server`  

Sets the size of the buffer used for proxying. By default, the buffer size is equal to one memory page. Depending on a platform, it is either 4K or 8K.

### `proxy_pass_error_message`

**Syntax**: `proxy_pass_error_message on | off ;`  
**Default**: `proxy_pass_error_message off;`  
**Context**: `mail , server`  

Indicates whether to pass the error message obtained during the authentication on the backend to the client. Usually, if the authentication in nginx is a success, the backend cannot return an error. If it nevertheless returns an error, it means some internal error has occurred. In such case the backend message can contain information that should not be shown to the client. However, responding with an error for the correct password is a normal behavior for some POP3 servers. For example, CommuniGatePro informs a user about mailbox overflow or other events by periodically outputting the authentication error . The directive should be enabled in this case.

### `proxy_timeout`

**Syntax**: `proxy_timeout timeout ;`  
**Default**: `proxy_timeout 24h;`  
**Context**: `mail , server`  

Sets the timeout between two successive read or write operations on client or proxied server connections. If no data is transmitted within this time, the connection is closed.

### `xclient`

**Syntax**: `xclient on | off ;`  
**Default**: `xclient on;`  
**Context**: `mail , server`  

Enables or disables the passing of the XCLIENT command with client parameters when connecting to the SMTP backend. With XCLIENT , the MTA is able to write client information to the log and apply various limitations based on this data. If XCLIENT is enabled then nginx passes the following commands when connecting to the backend:
