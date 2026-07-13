# Module ngx_otel_module

**Source**: https://nginx.org/en/docs/ngx_otel_module.html  
**Note**: 🔐 Requires commercial subscription (NGINX Plus)  

## Overview

Example Configuration Directives otel_exporter otel_service_name otel_resource_attr otel_trace otel_trace_context otel_span_name otel_span_attr Default span attributes Embedded Variables The ngx_otel_module module provides OpenTelemetry distributed tracing support. The module supports W3C context propagation and OTLP/gRPC export protocol. The source code of the module is available here . Download and install instructions are available here . The module is also available in a prebuilt nginx-module-otel package since 1.25.3 and in nginx-plus-module-otel package as part of our commercial subscription since 1.23.4.

## Example Configuration

```nginx
load_module modules/ngx_otel_module.so;

events {
}

http {

    otel_exporter {
        endpoint localhost:4317;
    }

    server {
        listen 127.0.0.1:8080;

        location / {
            otel_trace         on;
            otel_trace_context inject;

            proxy_pass http://backend;
        }
    }
}
```

## Directives

| Directive | Syntax | Default | Context |
|-----------|--------|---------|---------|
| `otel_exporter` | otel_exporter { ... } | — | http |
| `otel_service_name` | otel_service_name name ; | otel_service_name unknown_service:nginx; | http |
| `otel_trace` | otel_trace on \| off \| $variable ; | otel_trace off; | http , server , location |
| `otel_trace_context` | otel_trace_context extract \| inject \| propagate \| ignore ; | otel_trace_context ignore; | http , server , location |
| `otel_span_name` | otel_span_name name ; | — | http , server , location |
| `otel_span_attr` | otel_span_attr name value ; | — | http , server , location |

## Directive Details

### `otel_exporter`

**Syntax**: `otel_exporter { ... }`  
**Default**: `—`  
**Context**: `http`  

Specifies OTel data export parameters: Example: otel_exporter { endpoint https://otel-example.nginx.com:4317; header X-API-Token "my-token-value"; }

### `otel_service_name`

**Syntax**: `otel_service_name name ;`  
**Default**: `otel_service_name unknown_service:nginx;`  
**Context**: `http`  

Sets the “ service.name ” attribute of the OTel resource.

### `otel_trace`

**Syntax**: `otel_trace on | off | $variable ;`  
**Default**: `otel_trace off;`  
**Context**: `http , server , location`  

Enables or disables OpenTelemetry tracing. The directive can also be enabled by specifying a variable: split_clients "$otel_trace_id" $ratio_sampler { 10% on; * off; } server { location / { otel_trace $ratio_sampler; otel_trace_context inject; proxy_pass http://backend; } }

### `otel_trace_context`

**Syntax**: `otel_trace_context extract | inject | propagate | ignore ;`  
**Default**: `otel_trace_context ignore;`  
**Context**: `http , server , location`  

Specifies how to propagate traceparent/tracestate headers:

### `otel_span_name`

**Syntax**: `otel_span_name name ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Defines the name of the OTel span . By default, it is a name of the location for a request. The name can contain variables.

### `otel_span_attr`

**Syntax**: `otel_span_attr name value ;`  
**Default**: `—`  
**Context**: `http , server , location`  

Adds a custom OTel span attribute. The value can contain variables.
