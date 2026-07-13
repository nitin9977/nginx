# Module ngx_http_xslt_module

**Source**: https://nginx.org/en/docs/http/ngx_http_xslt_module.html  
**Note**: ⚠️ Not built by default — requires `--with-...` compile flag  

## Overview

Example Configuration Directives xml_entities xslt_last_modified xslt_param xslt_string_param xslt_stylesheet xslt_types The ngx_http_xslt_module (0.7.8+) is a filter that transforms XML responses using one or more XSLT stylesheets. This module is not built by default, it should be enabled with the --with-http_xslt_module configuration parameter. This module requires the libxml2 and libxslt libraries.

## Example Configuration

```nginx
location / {
    xml_entities    /site/dtd/entities.dtd;
    xslt_stylesheet /site/xslt/one.xslt param=value;
    xslt_stylesheet /site/xslt/two.xslt;
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `xml_entities` | xml_entities path ; | — | http , server , location |
| `xslt_stylesheet` | xslt_stylesheet stylesheet [ parameter = value ...]; | — | location |
| `xslt_types` | xslt_types mime-type ...; | xslt_types text/xml; | http , server , location |

## Directive Details

### `xml_entities`

**Syntax**: `xml_entities path ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Specifies the DTD file that declares character entities. This file is compiled at the configuration stage. For technical reasons, the module is unable to use the external subset declared in the processed XML, so it is ignored and a specially defined file is used instead. This file should not describe the XML structure. It is enough to declare just the required character entities, for example: <!ENTITY nbsp "&#xa0;">

### `xslt_stylesheet`

**Syntax**: `xslt_stylesheet stylesheet [ parameter = value ...];`  
**Default**: `—`  
**Context**: `location`  

Defines the XSLT stylesheet and its optional parameters. A stylesheet is compiled at the configuration stage. Parameters can either be specified separately, or grouped in a single line using the “ : ” delimiter. If a parameter includes the “ : ” character, it should be escaped as “ %3A ”. Also, libxslt requires to enclose parameters that contain non-alphanumeric characters into single or double quotes, for example: param1='http%3A//www.example.com':param2=value2

### `xslt_types`

**Syntax**: `xslt_types mime-type ...;`  
**Default**: `xslt_types text/xml;`  
**Context**: `http , server , location`  

Enables transformations in responses with the specified MIME types in addition to “ text/xml ”. The special value “ * ” matches any MIME type (0.8.29). If the transformation result is an HTML response, its MIME type is changed to “ text/html ”.
