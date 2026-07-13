# Module ngx_mail_imap_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_imap_module.html  

## Overview

Directives imap_auth imap_capabilities imap_client_buffer

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `imap_auth` | imap_auth method ...; | imap_auth plain; | mail , server |
| `imap_capabilities` | imap_capabilities extension ...; | imap_capabilities IMAP4 IMAP4rev1 UIDPLUS; | mail , server |
| `imap_client_buffer` | imap_client_buffer size ; | imap_client_buffer 4k\|8k; | mail , server |

## Directive Details

### `imap_auth`

**Syntax**: `imap_auth method ...;`  
**Default**: `imap_auth plain;`  
**Context**: `mail , server`  

Sets permitted methods of authentication for IMAP clients. Supported methods are: Plain text authentication methods (the LOGIN command, AUTH=PLAIN , and AUTH=LOGIN ) are always enabled, though if the plain and login methods are not specified, AUTH=PLAIN and AUTH=LOGIN will not be automatically included in imap_capabilities .

### `imap_capabilities`

**Syntax**: `imap_capabilities extension ...;`  
**Default**: `imap_capabilities IMAP4 IMAP4rev1 UIDPLUS;`  
**Context**: `mail , server`  

Sets the IMAP protocol extensions list that is passed to the client in response to the CAPABILITY command. The authentication methods specified in the imap_auth directive and STARTTLS are automatically added to this list depending on the starttls directive value. It makes sense to specify the extensions supported by the IMAP backends to which the clients are proxied (if these extensions are related to commands used after the authentication, when nginx transparently proxies a client connection to the backend). The current list of standardized extensions is published at www.iana.org .

### `imap_client_buffer`

**Syntax**: `imap_client_buffer size ;`  
**Default**: `imap_client_buffer 4k|8k;`  
**Context**: `mail , server`  

Sets the size of the buffer used for reading IMAP commands. By default, the buffer size is equal to one memory page. This is either 4K or 8K, depending on a platform.
