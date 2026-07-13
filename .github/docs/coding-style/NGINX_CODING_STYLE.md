# nginx Coding Style Guide

> **Source**: [nginx Development Guide — Code style](https://nginx.org/en/docs/dev/development_guide.html#code_style)
>
> This document is the authoritative style reference for all nginx C source code
> produced or reviewed by the developer and principal agents. Every rule below
> is mandatory unless explicitly marked optional.

---

## Table of Contents

1. [General Rules](#1-general-rules)
2. [Files](#2-files)
3. [Comments](#3-comments)
4. [Preprocessor](#4-preprocessor)
5. [Types](#5-types)
6. [Variables](#6-variables)
7. [Functions](#7-functions)
8. [Expressions](#8-expressions)
9. [Conditionals and Loops](#9-conditionals-and-loops)
10. [Labels](#10-labels)
11. [Module Structure Patterns](#11-module-structure-patterns)
12. [Configuration Directives](#12-configuration-directives)
13. [Variable Handlers](#13-variable-handlers)
14. [Error Handling](#14-error-handling)
15. [Memory Management](#15-memory-management)
16. [Common Pitfalls](#16-common-pitfalls)

---

## 1. General Rules

- Maximum text width is **80 characters**.
- Indentation is **4 spaces**. No tabs.
- No trailing spaces.
- List elements on the same line are separated with spaces.
- Hexadecimal literals are **lowercase**.
- File names, function and type names, and global variables have the `ngx_` or
  more specific prefix such as `ngx_http_` and `ngx_mail_`.

```c
size_t
ngx_utf8_length(u_char *p, size_t n)
{
    u_char  c, *last;
    size_t  len;

    last = p + n;

    for (len = 0; p < last; len++) {

        c = *p;

        if (c < 0x80) {
            p++;
            continue;
        }

        if (ngx_utf8_decode(&p, last - p) > 0x10ffff) {
            /* invalid UTF-8 */
            return n;
        }
    }

    return len;
}
```

---

## 2. Files

A typical source file contains the following sections separated by **two empty lines**:

1. Copyright statements
2. Includes
3. Preprocessor definitions
4. Type definitions
5. Function prototypes
6. Variable definitions
7. Function definitions

### Copyright

```c
/*
 * Copyright (C) Author Name
 * Copyright (C) Organization, Inc.
 */
```

If the file is modified significantly, the list of authors should be updated;
the new author is added to the top.

### Include Order

`ngx_config.h` and `ngx_core.h` are always included first, followed by one of
`ngx_http.h`, `ngx_stream.h`, or `ngx_mail.h`. Then follow optional external
header files:

```c
#include <ngx_config.h>
#include <ngx_core.h>
#include <ngx_http.h>

#include <libxml/parser.h>
#include <libxml/tree.h>
#include <libxslt/xslt.h>

#if (NGX_HAVE_EXSLT)
#include <libexslt/exslt.h>
#endif
```

### Header Protection

```c
#ifndef _NGX_PROCESS_CYCLE_H_INCLUDED_
#define _NGX_PROCESS_CYCLE_H_INCLUDED_
...
#endif /* _NGX_PROCESS_CYCLE_H_INCLUDED_ */
```

---

## 3. Comments

- `//` comments are **not used** — always use `/* ... */`.
- Text is written in English; American spelling is preferred.
- Multi-line comments:

```c
/*
 * The red-black tree code is based on the algorithm described in
 * the "Introduction to Algorithms" by Cormen, Leiserson and Rivest.
 */
```

- Single-line comments:

```c
/* find the server configuration for the address:port */
```

---

## 4. Preprocessor

- Macro names start with `ngx_` or `NGX_` (or more specific) prefix.
- Constants are **UPPERCASE**. Parameterized macros and initializer macros are **lowercase**.
- The macro name and value are separated by **at least two spaces**.

```c
#define NGX_CONF_BUFFER  4096

#define ngx_buf_in_memory(b)  (b->temporary || b->memory || b->mmap)

#define ngx_buf_size(b)                                                      \
    (ngx_buf_in_memory(b) ? (off_t) (b->last - b->pos):                     \
                            (b->file_last - b->file_pos))

#define ngx_null_string  { 0, NULL }
```

- Conditions are inside parentheses; negation is outside:

```c
#if (NGX_HAVE_KQUEUE)
...
#elif ((NGX_HAVE_DEVPOLL && !(NGX_TEST_BUILD_DEVPOLL)) \
       || (NGX_HAVE_EVENTPORT && !(NGX_TEST_BUILD_EVENTPORT)))
...
#endif /* NGX_HAVE_KQUEUE */
```

---

## 5. Types

- Type names end with `_t`. A defined type name is separated by **at least two spaces**.

```c
typedef ngx_uint_t  ngx_rbtree_key_t;
```

- Structure types are defined using `typedef`. Inside structures, member types
  and names are **aligned**:

```c
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

- Keep alignment identical among different structures in the file.
- A structure that points to itself has the name ending with `_s`:

```c
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};
```

- Adjacent structure definitions are separated with **two empty lines**.
- Each structure member is declared on its own line.
- Function pointers inside structures have defined types ending with `_pt`:

```c
typedef ssize_t (*ngx_recv_pt)(ngx_connection_t *c, u_char *buf, size_t size);
```

- Enumerations have types ending with `_e`:

```c
typedef enum {
    ngx_http_fastcgi_st_version = 0,
    ngx_http_fastcgi_st_type,
    ...
    ngx_http_fastcgi_st_padding
} ngx_http_fastcgi_state_e;
```

---

## 6. Variables

Variables are declared **sorted by length of base type, then alphabetically**.
Type names and variable names are aligned. The type and name "columns" are
separated with **two spaces**. Large arrays are put at the end of a
declaration block:

```c
u_char                      *rv, *p;
ngx_conf_t                  *cf;
ngx_uint_t                   i, j, k;
unsigned int                 len;
struct sockaddr             *sa;
const unsigned char         *data;
ngx_peer_connection_t       *pc;
ngx_http_core_srv_conf_t   **cscfp;
ngx_http_upstream_srv_conf_t *us, *uscf;
u_char                       text[NGX_SOCKADDR_STRLEN];
```

### Common Type/Name Conventions

```c
u_char                        *rv;
ngx_int_t                      rc;
ngx_conf_t                    *cf;
ngx_connection_t              *c;
ngx_http_request_t            *r;
ngx_peer_connection_t         *pc;
ngx_http_upstream_srv_conf_t  *us, *uscf;
```

### Static and Global Variables

May be initialized on declaration:

```c
static ngx_str_t  ngx_http_memcached_key = ngx_string("memcached_key");
```

```c
static uint32_t  ngx_crc32_table16[] = {
    0x00000000, 0x1db71064, 0x3b6e20c8, 0x26d930ac,
    ...
    0x9b64c2b0, 0x86d3d2d4, 0xa00ae278, 0xbdbdf21c
};
```

---

## 7. Functions

- **All functions** (even `static` ones) should have prototypes.
- Prototypes include argument names.
- Long prototypes are wrapped with a **single indentation** on continuation lines:

```c
static char *ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
static ngx_int_t ngx_http_init_phases(ngx_conf_t *cf,
    ngx_http_core_main_conf_t *cmcf);

static char *ngx_http_merge_servers(ngx_conf_t *cf,
    ngx_http_core_main_conf_t *cmcf, ngx_http_module_t *module,
    ngx_uint_t ctx_index);
```

- The **function name in a definition starts on a new line**:

```c
static ngx_int_t
ngx_http_find_virtual_server(ngx_http_request_t *r, u_char *host, size_t len)
{
    ...
}
```

- Function body opening and closing braces are on **separate lines**.
- There are **two empty lines** between functions.
- No space after the function name and opening parenthesis.
- Long function calls are wrapped so continuation lines start from the position
  of the first function argument:

```c
ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
               "http header: \"%V: %V\"",
               &h->key, &h->value);
```

- If impossible, format so the first continuation line ends at position 79:

```c
hc->busy = ngx_palloc(r->connection->pool,
                  cscf->large_client_header_buffers.num * sizeof(ngx_buf_t *));
```

- Use `ngx_inline` instead of `inline`:

```c
static ngx_inline void ngx_cpuid(uint32_t i, uint32_t *buf);
```

---

## 8. Expressions

- Binary operators except `.` and `->` are separated from operands by one space.
- Unary operators and subscripts are NOT separated from operands.

```c
width = width * 10 + (*fmt++ - '0');
ch = (u_char) ((decoded << 4) + (ch - '0'));
r->exten.data = &r->uri.data[i + 1];
```

- Type casts are separated by **one space** from the casted expression.
- An asterisk inside a type cast is separated with a space from the type name:

```c
len = ngx_sock_ntop((struct sockaddr *) sin6, p, len, 1);
```

- Long expressions wrap at binary operators. The continuation line is lined up
  with the start of expression:

```c
if (status == NGX_HTTP_MOVED_PERMANENTLY
    || status == NGX_HTTP_MOVED_TEMPORARILY
    || status == NGX_HTTP_SEE_OTHER
    || status == NGX_HTTP_TEMPORARY_REDIRECT
    || status == NGX_HTTP_PERMANENT_REDIRECT)
{
    ...
}
```

- String literal concatenation:

```c
p->temp_file->warn = "an upstream response is buffered "
                     "to a temporary file";
```

- Pointers are explicitly compared to `NULL` (not `0`):

```c
if (ptr != NULL) {
    ...
}
```

---

## 9. Conditionals and Loops

### `if` Statements

- One space between `if` and the condition.
- Opening brace on the same line (or on a dedicated line if the condition
  takes several lines).
- Usually an empty line before `else if` / `else`:

```c
if (node->left == sentinel) {
    temp = node->right;
    subst = node;

} else if (node->right == sentinel) {
    temp = node->left;
    subst = node;

} else {
    subst = ngx_rbtree_min(node->right, sentinel);

    if (subst->left != sentinel) {
        temp = subst->left;

    } else {
        temp = subst->right;
    }
}
```

### `while` and `do...while`

```c
while (p < last && *p == ' ') {
    p++;
}

do {
    ctx->node = rn;
    ctx = ctx->next;
} while (ctx);
```

### `switch`

- One space between `switch` and the condition.
- Opening brace on the same line.
- `case` keywords are lined up with `switch`:

```c
switch (ch) {
case '!':
    looked = 2;
    state = ssi_comment0_state;
    break;

case '<':
    copy_end = p;
    break;

default:
    copy_end = p;
    looked = 0;
    state = ssi_start_state;
    break;
}
```

### `for` Loops

```c
for (i = 0; i < ccf->env.nelts; i++) {
    ...
}

for (q = ngx_queue_head(locations);
     q != ngx_queue_sentinel(locations);
     q = ngx_queue_next(q))
{
    ...
}
```

- Omitted parts use `/* void */`:

```c
for (i = 0; /* void */ ; i++) {
    ...
}
```

- Empty loop body:

```c
for (cl = *busy; cl->next; cl = cl->next) { /* void */ }
```

- Endless loop:

```c
for ( ;; ) {
    ...
}
```

---

## 10. Labels

Labels are surrounded with empty lines and indented at the previous level:

```c
    if (i == 0) {
        u->err = "host not found";
        goto failed;
    }

    u->addrs = ngx_pcalloc(pool, i * sizeof(ngx_addr_t));
    if (u->addrs == NULL) {
        goto failed;
    }

    u->naddrs = i;

    ...

    return NGX_OK;

failed:

    freeaddrinfo(res);
    return NGX_ERROR;
```

---

## 11. Module Structure Patterns

### Complete Module Skeleton

```c
/*
 * Copyright (C) Author.
 */


#include <ngx_config.h>
#include <ngx_core.h>


typedef struct {
    ngx_flag_t  enable;
} ngx_foo_conf_t;


static void *ngx_foo_create_conf(ngx_cycle_t *cycle);
static char *ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf);

static char *ngx_foo_enable(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_enable_post = { ngx_foo_enable };


static ngx_command_t  ngx_foo_commands[] = {

    { ngx_string("foo_enabled"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_foo_conf_t, enable),
      &ngx_foo_enable_post },

      ngx_null_command
};


static ngx_core_module_t  ngx_foo_module_ctx = {
    ngx_string("foo"),
    ngx_foo_create_conf,
    ngx_foo_init_conf
};


ngx_module_t  ngx_foo_module = {
    NGX_MODULE_V1,
    &ngx_foo_module_ctx,                   /* module context */
    ngx_foo_commands,                      /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

### HTTP Module Context Pattern

```c
static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_foo_create_loc_conf,          /* create location configuration */
    ngx_http_foo_merge_loc_conf            /* merge location configuration */
};
```

### Config Merge Pattern

```c
static char *
ngx_http_foo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_foo_loc_conf_t *prev = parent;
    ngx_http_foo_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf->foo, prev->foo, 1);

    return NGX_CONF_OK;
}
```

---

## 12. Configuration Directives

### `ngx_command_t` Fields

| Field | Description |
|-------|-------------|
| `name` | Directive name (`ngx_string(...)`) |
| `type` | Bit-field: context flags + argument count flags |
| `set` | Handler function pointer |
| `conf` | Config offset (`NGX_HTTP_MAIN_CONF_OFFSET`, etc.) |
| `offset` | Field offset within the config struct (`offsetof(...)`) |
| `post` | Post-handler (validation callback) |

### Argument Flags

| Flag | Meaning |
|------|---------|
| `NGX_CONF_NOARGS` | No arguments |
| `NGX_CONF_TAKE1` .. `NGX_CONF_TAKE7` | Exact argument count |
| `NGX_CONF_1MORE` | One or more arguments |
| `NGX_CONF_FLAG` | Boolean `on`/`off` value |
| `NGX_CONF_BLOCK` | Block directive |

### Context Flags

`NGX_MAIN_CONF`, `NGX_HTTP_MAIN_CONF`, `NGX_HTTP_SRV_CONF`, `NGX_HTTP_LOC_CONF`,
`NGX_STREAM_MAIN_CONF`, `NGX_STREAM_SRV_CONF`, `NGX_MAIL_MAIN_CONF`, etc.

### Built-in `set` Handlers

`ngx_conf_set_flag_slot`, `ngx_conf_set_str_slot`, `ngx_conf_set_num_slot`,
`ngx_conf_set_size_slot`, `ngx_conf_set_off_slot`, `ngx_conf_set_msec_slot`,
`ngx_conf_set_sec_slot`, `ngx_conf_set_bufs_slot`, `ngx_conf_set_enum_slot`,
`ngx_conf_set_bitmask_slot`.

### Terminate with `ngx_null_command`

```c
static ngx_command_t  ngx_http_foo_commands[] = {

    { ngx_string("foo"),
      NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_str_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_foo_loc_conf_t, value),
      NULL },

      ngx_null_command
};
```

---

## 13. Variable Handlers

### Registering Variables (preconfiguration)

```c
static ngx_http_variable_t  ngx_http_foo_vars[] = {

    { ngx_string("foo_v1"), NULL, ngx_http_foo_v1_variable, 0, 0, 0 },

      ngx_http_null_variable
};

static ngx_int_t
ngx_http_foo_add_variables(ngx_conf_t *cf)
{
    ngx_http_variable_t  *var, *v;

    for (v = ngx_http_foo_vars; v->name.len; v++) {
        var = ngx_http_add_variable(cf, &v->name, v->flags);
        if (var == NULL) {
            return NGX_ERROR;
        }

        var->get_handler = v->get_handler;
        var->data = v->data;
    }

    return NGX_OK;
}
```

### Get Handler Pattern

```c
static ngx_int_t
ngx_http_variable_connection(ngx_http_request_t *r,
    ngx_http_variable_value_t *v, uintptr_t data)
{
    u_char  *p;

    p = ngx_pnalloc(r->pool, NGX_ATOMIC_T_LEN);
    if (p == NULL) {
        return NGX_ERROR;
    }

    v->len = ngx_sprintf(p, "%uA", r->connection->number) - p;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;
    v->data = p;

    return NGX_OK;
}
```

### Runtime Access

```c
ngx_http_variable_value_t  *v;

v = ngx_http_get_flushed_variable(r, index);

if (v == NULL || v->not_found) {
    /* variable not found */
    return NGX_ERROR;
}
```

---

## 14. Error Handling

### Common Return Codes

| Code | Meaning |
|------|---------|
| `NGX_OK` | Operation succeeded |
| `NGX_ERROR` | Operation failed |
| `NGX_AGAIN` | Incomplete; call again |
| `NGX_DECLINED` | Rejected (not an error) |
| `NGX_BUSY` | Resource unavailable |
| `NGX_DONE` | Complete or continued elsewhere |
| `NGX_ABORT` | Function aborted |

### Error Number Handling

Store `ngx_errno` in a local `ngx_err_t` variable if accessing more than once:

```c
ngx_int_t
ngx_my_kill(ngx_pid_t pid, ngx_log_t *log, int signo)
{
    ngx_err_t  err;

    if (kill(pid, signo) == -1) {
        err = ngx_errno;

        ngx_log_error(NGX_LOG_ALERT, log, err,
                      "kill(%P, %d) failed", pid, signo);

        if (err == NGX_ESRCH) {
            return 2;
        }

        return 1;
    }

    return 0;
}
```

### nginx Formatting Extensions

| Format | Type |
|--------|------|
| `%O` | `off_t` |
| `%T` | `time_t` |
| `%z` | `ssize_t` |
| `%i` | `ngx_int_t` |
| `%p` | `void *` |
| `%V` | `ngx_str_t *` |
| `%s` | `u_char *` (null-terminated) |
| `%*s` | `size_t + u_char *` |

Prepend `u` for unsigned. Use `X` or `x` for hex output.

---

## 15. Memory Management

### Pool Operations

| Function | Purpose |
|----------|---------|
| `ngx_create_pool(size, log)` | Create pool with specified block size |
| `ngx_destroy_pool(pool)` | Free all pool memory including the pool |
| `ngx_palloc(pool, size)` | Allocate aligned memory |
| `ngx_pcalloc(pool, size)` | Allocate aligned memory, zero-filled |
| `ngx_pnalloc(pool, size)` | Allocate unaligned memory (for strings) |
| `ngx_pfree(pool, p)` | Free large allocations only |

### Pool Cleanup Pattern

```c
ngx_pool_cleanup_t  *cln;

cln = ngx_pool_cleanup_add(pool, 0);
if (cln == NULL) { /* error */ }

cln->handler = ngx_my_cleanup;
cln->data = "foo";

...

static void
ngx_my_cleanup(void *data)
{
    u_char  *msg = data;

    ngx_do_smth(msg);
}
```

### Memory Rules

- Pools are scoped lifetimes: **cycle > connection > request**.
- When the object is destroyed, the associated pool is destroyed too.
- Always allocate from the appropriate pool, not `malloc()`/`free()`.
- Check every allocation for `NULL` return.

---

## 16. Common Pitfalls

### Writing a C module

Consider whether the feature can be implemented with existing modules or
built-in scripting (Perl, njs) before writing a new C module. If a module is
unavoidable, keep it as small and simple as possible.

### C Strings

`ngx_str_t` is **not** a NUL-terminated C string. Do not pass `data` to
`strlen()`, `strstr()`, or other standard C string functions. Use nginx's
equivalents (`ngx_strlen()`, `ngx_strstr()`, etc.) or pass `data + len`.
Exception: strings from configuration file parsing are NUL-terminated.

### Global Variables

Avoid global variables. All data should be tied to a configuration cycle and
allocated from the corresponding pool. Global variables break graceful config
reloads because two configurations cannot coexist.

### Manual Memory Management

Do not use `malloc()`/`free()`. Use nginx pools. A pool is tied to an object
(configuration, cycle, connection, request) and is destroyed automatically
when the object is destroyed.

### Threads

Avoid threads. Most nginx functions are not thread-safe. If code is not related
to client request processing, schedule a timer in `init_process` and perform
actions in the timer handler.

### Blocking Libraries

Use only libraries with asynchronous interfaces. Synchronous/blocking libraries
will block the entire nginx worker, destroying performance.

### HTTP Requests to External Services

Do not use external HTTP libraries (e.g., libcurl). Use nginx's subrequest API
for request-context calls, or the built-in HTTP client for worker-context calls
(e.g., OCSP module pattern).
