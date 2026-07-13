# Module ngx_mail_smtp_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_smtp_module.html  

## Overview

Directives smtp_auth smtp_capabilities smtp_client_buffer smtp_greeting_delay

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `smtp_auth` | smtp_auth method ...; | smtp_auth plain login; | mail , server |
| `smtp_capabilities` | smtp_capabilities extension ...; | — | mail , server |
| `smtp_client_buffer` | smtp_client_buffer size ; | smtp_client_buffer 4k\|8k; | mail , server |
| `smtp_greeting_delay` | smtp_greeting_delay time ; | smtp_greeting_delay 0; | mail , server |

## Directive Details

### `smtp_auth`

**Syntax**: `smtp_auth method ...;`  
**Default**: `smtp_auth plain login;`  
**Context**: `mail , server`  

Sets permitted methods of SASL authentication for SMTP clients. Supported methods are: Plain text authentication methods ( AUTH PLAIN and AUTH LOGIN ) are always enabled, though if the plain and login methods are not specified, AUTH PLAIN and AUTH LOGIN will not be automatically included in smtp_capabilities .

### `smtp_capabilities`

**Syntax**: `smtp_capabilities extension ...;`  
**Default**: `—`  
**Context**: `mail , server`  

Sets the SMTP protocol extensions list that is passed to the client in response to the EHLO command. The authentication methods specified in the smtp_auth directive and STARTTLS are automatically added to this list depending on the starttls directive value. It makes sense to specify the extensions supported by the MTA to which the clients are proxied (if these extensions are related to commands used after the authentication, when nginx transparently proxies the client connection to the backend). The current list of standardized extensions is published at www.iana.org .

### `smtp_client_buffer`

**Syntax**: `smtp_client_buffer size ;`  
**Default**: `smtp_client_buffer 4k|8k;`  
**Context**: `mail , server`  

Sets the size of the buffer used for reading SMTP commands. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.

### `smtp_greeting_delay`

**Syntax**: `smtp_greeting_delay time ;`  
**Default**: `smtp_greeting_delay 0;`  
**Context**: `mail , server`  

Allows setting a delay before sending an SMTP greeting in order to reject clients who fail to wait for the greeting before sending SMTP commands.
