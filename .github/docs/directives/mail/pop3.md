# Module ngx_mail_pop3_module

**Source**: https://nginx.org/en/docs/mail/ngx_mail_pop3_module.html  

## Overview

Directives pop3_auth pop3_capabilities

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `pop3_auth` | pop3_auth method ...; | pop3_auth plain; | mail , server |
| `pop3_capabilities` | pop3_capabilities extension ...; | pop3_capabilities TOP USER UIDL; | mail , server |

## Directive Details

### `pop3_auth`

**Syntax**: `pop3_auth method ...;`  
**Default**: `pop3_auth plain;`  
**Context**: `mail , server`  

Sets permitted methods of authentication for POP3 clients. Supported methods are: Plain text authentication methods ( USER/PASS , AUTH PLAIN , and AUTH LOGIN ) are always enabled, though if the plain method is not specified, AUTH PLAIN and AUTH LOGIN will not be automatically included in pop3_capabilities .

### `pop3_capabilities`

**Syntax**: `pop3_capabilities extension ...;`  
**Default**: `pop3_capabilities TOP USER UIDL;`  
**Context**: `mail , server`  

Sets the POP3 protocol extensions list that is passed to the client in response to the CAPA command. The authentication methods specified in the pop3_auth directive ( SASL extension) and STLS are automatically added to this list depending on the starttls directive value. It makes sense to specify the extensions supported by the POP3 backends to which the clients are proxied (if these extensions are related to commands used after the authentication, when nginx transparently proxies the client connection to the backend). The current list of standardized extensions is published at www.iana.org .
