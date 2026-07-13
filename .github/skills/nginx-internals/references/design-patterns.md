# Design Patterns — nginx Module & Feature Development

> **Source paths**: All `src/` paths below are relative to the **nginx repository root** (i.e. `<repo>/src/`), not relative to `.github/`.
>
> **Coding Style**: All code examples in this file and all code produced using these patterns
> **must** conform to the [nginx Coding Style Guide](../../../docs/coding-style/NGINX_CODING_STYLE.md),
> which is derived from the official [nginx development guide](https://nginx.org/en/docs/dev/development_guide.html#code_style).

Patterns for implementing new modules, phase handlers, config directives, and features within nginx conventions.

---

## 1. Module Registration Pattern

Every nginx module follows this skeleton:

```c
/* 1. Config struct */
typedef struct {
    ngx_flag_t  enable;
    ngx_str_t   setting;
} ngx_http_mymodule_conf_t;

/* 2. Directive array */
static ngx_command_t  ngx_http_mymodule_commands[] = {

    { ngx_string("my_directive"),
      NGX_HTTP_LOC_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_mymodule_conf_t, enable),
      NULL },

      ngx_null_command
};

/* 3. Module context (config lifecycle) */
static ngx_http_module_t  ngx_http_mymodule_ctx = {
    NULL,                               /* preconfiguration */
    ngx_http_mymodule_postconfiguration,/* postconfiguration */
    NULL,                               /* create main conf */
    NULL,                               /* init main conf */
    NULL,                               /* create srv conf */
    NULL,                               /* merge srv conf */
    ngx_http_mymodule_create_loc_conf,  /* create loc conf */
    ngx_http_mymodule_merge_loc_conf    /* merge loc conf */
};

/* 4. Module definition */
ngx_module_t  ngx_http_mymodule_module = {
    NGX_MODULE_V1,
    &ngx_http_mymodule_ctx,
    ngx_http_mymodule_commands,
    NGX_HTTP_MODULE,
    NULL, NULL, NULL, NULL, NULL, NULL, NULL,
    NGX_MODULE_V1_PADDING
};
```

**Key callbacks**:
- `create_*_conf`: Allocate config struct from `cf->pool`, set defaults to `NGX_CONF_UNSET`
- `merge_*_conf`: Inherit parent values with `ngx_conf_merge_*_value()` macros
- `postconfiguration`: Register phase handlers or filters

## 2. Phase Handler Pattern (HTTP)

Register in `postconfiguration`:

```c
static ngx_int_t
ngx_http_mymodule_postconfiguration(ngx_conf_t *cf)
{
    ngx_http_core_main_conf_t  *cmcf;
    ngx_http_handler_pt        *h;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_mymodule_handler;

    return NGX_OK;
}
```

**Handler return codes** (for non-content phases):
- `NGX_OK` → access granted, skip remaining handlers in this phase
- `NGX_DECLINED` → pass to next handler in same phase
- `NGX_HTTP_FORBIDDEN` (or other status) → terminate with error
- `NGX_AGAIN` / `NGX_DONE` → async wait, resume later

**Content handler**: Set `r->content_handler` or `clcf->handler` instead of registering in phase array.

## 3. Phase Handler Pattern (Stream)

```c
static ngx_int_t
ngx_stream_mymodule_postconfiguration(ngx_conf_t *cf)
{
    ngx_stream_core_main_conf_t  *cmcf;
    ngx_stream_handler_pt        *h;

    cmcf = ngx_stream_conf_get_module_main_conf(cf,
                                                ngx_stream_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_STREAM_ACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_stream_mymodule_handler;

    return NGX_OK;
}
```

**Stream checker semantics** (different from HTTP):
- Generic checker: `NGX_OK` → skip to `ph->next`, `NGX_DECLINED` → `phase_handler++`
- Preread checker: `NGX_AGAIN` → arm timer + read event, return to event loop
- Content checker: calls `cscf->handler(s)` directly, returns `NGX_OK` always

## 4. Filter Chain Pattern (HTTP)

**Header filter**:
```c
static ngx_http_output_header_filter_pt ngx_http_next_header_filter;

static ngx_int_t
ngx_http_mymodule_header_filter(ngx_http_request_t *r)
{
    /* modify headers */
    r->headers_out.content_type_len = sizeof("text/plain") - 1;
    return ngx_http_next_header_filter(r);
}

/* In postconfiguration: */
ngx_http_next_header_filter = ngx_http_top_header_filter;
ngx_http_top_header_filter = ngx_http_mymodule_header_filter;
```

**Body filter**: Same pattern with `ngx_http_top_body_filter`. Process `ngx_chain_t` buffers, pass remainder to next filter.

## 5. Config Directive Patterns

**Common `set` helpers** (avoid writing custom parsers):

| Helper | Directive Type | Example |
|--------|---------------|---------|
| `ngx_conf_set_flag_slot` | `on/off` | `my_enable on;` |
| `ngx_conf_set_str_slot` | string | `my_path /tmp;` |
| `ngx_conf_set_num_slot` | integer | `my_count 10;` |
| `ngx_conf_set_size_slot` | size (k/m/g) | `my_size 4k;` |
| `ngx_conf_set_msec_slot` | time (ms/s/m/h) | `my_timeout 30s;` |
| `ngx_conf_set_enum_slot` | enum values | `my_mode auto;` |
| `ngx_conf_set_bitmask_slot` | bitmask | `my_flags a b;` |

**Custom directive handler**: For complex directives (blocks, multi-value):
```c
static char *
ngx_http_mymodule_directive(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_str_t  *value = cf->args->elts;
    /* value[0] = directive name, value[1..n] = arguments */
    /* return NGX_CONF_OK or NGX_CONF_ERROR */
}
```

## 6. Memory Allocation Patterns

**Rule**: Always allocate from a pool with appropriate lifetime.

| Lifetime | Pool | Use For |
|----------|------|---------|
| Process | `cycle->pool` | Config structs, module state |
| Connection | `c->pool` | Per-connection data, session |
| Request | `r->pool` | Per-request data, variables |
| Temporary | `cf->pool` | Config parsing temporaries |
| Shared | `ngx_slab_alloc()` | Cross-worker state (rare) |

**Never** `malloc()` directly — use `ngx_alloc()` only for data that outlives all pools (extremely rare).

## 7. Timer Pattern

```c
/* arm timer */
ngx_add_timer(c->read, timeout_ms);

/* in handler, check timeout */
if (rev->timedout) {
    ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT,
                  "client timed out");
    /* finalize/close */
    return;
}

/* clear timer on success */
if (c->read->timer_set) {
    ngx_del_timer(c->read);
}
```

**Invariant**: Every `ngx_add_timer` must have a corresponding `ngx_del_timer` on the success path.

## 8. Upstream Module Pattern

```c
/* in content handler: */
ngx_http_upstream_t  *u;

if (ngx_http_upstream_create(r) != NGX_OK) {
    return NGX_HTTP_INTERNAL_SERVER_ERROR;
}

u = r->upstream;
u->conf = &mycf->upstream;
u->create_request = my_create_request;
u->process_header = my_process_header;
u->reinit_request = my_reinit_request;
u->finalize_request = my_finalize_request;

ngx_http_upstream_init(r);
return NGX_DONE;
```

## 9. Variable Registration Pattern

```c
static ngx_http_variable_t  ngx_http_mymodule_vars[] = {

    { ngx_string("my_var"), NULL,
      ngx_http_mymodule_variable, 0, NGX_HTTP_VAR_NOCACHEABLE, 0 },

      ngx_http_null_variable
};

/* in preconfiguration: */
for (v = ngx_http_mymodule_vars; v->name.len; v++) {
    var = ngx_http_add_variable(cf, &v->name, v->flags);
    var->get_handler = v->get_handler;
    var->data = v->data;
}
```

## 10. Feature Design Checklist

When designing a new feature:

- [ ] Which module type? (core / event / http / stream / mail)
- [ ] Which phase? (for request processing modules)
- [ ] Config scope? (main / server / location for HTTP)
- [ ] Memory lifetime? (cycle / connection / request pool)
- [ ] Async operations? (DNS, upstream connect, file I/O)
- [ ] Timer requirements? (timeout, periodic)
- [ ] Error handling? (which status codes, cleanup path)
- [ ] Reload safety? (does state survive config reload?)
- [ ] Worker isolation? (shared memory needed?)
- [ ] Filter or handler? (modify response vs generate response)
- [ ] Variable exposure? (does it export `$variables`?)
- [ ] Log phase integration? (custom access log fields?)
- [ ] Coding style? (all code follows [nginx Coding Style Guide](../../../docs/coding-style/NGINX_CODING_STYLE.md))
