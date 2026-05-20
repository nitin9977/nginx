# NGINX Misc Module Ramp-Up Guide

**Audience**: Engineers onboarding to `src/misc` internals  
**Scope**: Architecture, integration, and walkthrough for misc module behavior  
**Date**: 2026-05-19

---

## 1. Architecture and Design of `src/misc`

### 1.1 What `src/misc` contains

`src/misc` is not a cohesive runtime subsystem like core/event/http. It is a small holding area for special-purpose modules and compatibility helpers.

It contains two files:

- `ngx_google_perftools_module.c` — CPU profiling integration module
- `ngx_cpp_test_module.cpp` — C++ compatibility test stub for nginx headers

### 1.2 Design intent

- Keep optional tooling hooks out of critical request path code
- Allow build-time/runtime opt-in behavior for diagnostics
- Preserve header ABI compatibility checks with C++ compiler

### 1.3 Key design objects

#### `ngx_google_perftools_conf_t` (`src/misc/ngx_google_perftools_module.c`, line 25)

Simple config struct:

- `profiles`: `ngx_str_t` — configured path prefix for CPU profile output files

#### Module registration (`ngx_google_perftools_module`, line 52)

Registered as `NGX_CORE_MODULE` with:

- `module context`: `ngx_google_perftools_module_ctx` — provides `create_conf` callback, no `init_conf`
- `module directives`: single directive `google_perftools_profiles` — `NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_TAKE1`
- `init process`: `ngx_google_perftools_worker` — activation hook per worker

Lifecycle callbacks used: only `create_conf` and `init_process`. No `init_module`, `exit_process`, or master hooks.

#### External profiler API (`src/misc/ngx_google_perftools_module.c`, lines 14-17)

Three functions declared manually (avoiding C++ header include):

- `ProfilerStart(u_char *fname)` — start CPU profiling to file
- `ProfilerStop(void)` — stop profiling
- `ProfilerRegisterThread(void)` — register thread for signal-based sampling

#### `ngx_cpp_test_module.cpp`

C++ compilation unit that `#include`s all nginx headers under `extern "C"`. Contains:

- Empty `ngx_http_module_t` context and `ngx_module_t` definition
- Empty handler function returning `NGX_DECLINED`
- Purely build-validation — never loaded at runtime unless explicitly configured

### 1.4 Operational significance

- Google perftools module can affect worker runtime due to `ITIMER_PROF` timer activation
- Profile data is per-worker (one file per PID), avoiding write contention
- `ProfilerStop()` is called implicitly on profiler destruction (worker exit), not via `exit_process` hook
- cpp test module is inert at runtime — primarily validates header compatibility

---

## 2. Integration and Dependencies

### 2.1 Integration with `src/core` module framework

Google perftools module registers as a core module (`NGX_CORE_MODULE`):

- `create_conf` allocates `ngx_google_perftools_conf_t` from `cycle->pool`
- Config retrieved at runtime via `ngx_get_conf(cycle->conf_ctx, ngx_google_perftools_module)`
- Directive uses standard `ngx_conf_set_str_slot()` helper with `offsetof`

### 2.2 Integration with process model

Because the hook is `init_process`, each worker independently:

1. Reads its own copy of the config from `cycle->conf_ctx`
2. Produces independent profile files (path + `.pid` suffix)
3. Handles inherited profiler state from master process

This avoids write contention and attribution ambiguity across workers.

### 2.3 External dependency integration

Google perftools module directly references profiler symbols:

- `ProfilerStart`, `ProfilerStop`, `ProfilerRegisterThread`
- These are declared manually because the profiler header (`<google/profiler.h>`) is C++
- Link-time dependency: `-lprofiler` must be present

### 2.4 Environment interaction

- If `CPUPROFILE` environment variable is inherited from master, worker calls `ProfilerStop()` first to avoid inherited profiler state confusion
- Profile output path is built as: `<configured_prefix>.<pid>\0`
- Buffer allocated via `ngx_alloc()` with size `profiles.len + NGX_INT_T_LEN + 2`

---

## 3. Basic Code Walkthrough

### 3.1 Generic tracing template

For `src/misc`, the representative event is worker-process startup when profiling is enabled:

1. Process enters worker init callbacks
2. Module reads its config from `cycle->conf_ctx`
3. If disabled (empty path), no-op and return
4. If enabled, initialize profiler runtime and register thread

### 3.2 Concrete walkthrough: `ngx_google_perftools_worker()` (line 90)

#### Step A: load module config

```
gptcf = ngx_get_conf(cycle->conf_ctx, ngx_google_perftools_module);
if (gptcf->profiles.len == 0) {
    return NGX_OK;  // profiling not configured — no-op
}
```

#### Step B: allocate profile filename buffer

```
profile = ngx_alloc(gptcf->profiles.len + NGX_INT_T_LEN + 2, cycle->log);
```

Allocated via raw `malloc` (`ngx_alloc`), not pool — freed explicitly at end.

#### Step C: sanitize inherited profiling state

```
if (getenv("CPUPROFILE")) {
    ProfilerStop();  // stop profiler inherited from master
}
```

This prevents double-profiling when master process had `CPUPROFILE` set.

#### Step D: format filename and start profiler

```
ngx_sprintf(profile, "%V.%d%Z", &gptcf->profiles, ngx_pid);
if (ProfilerStart(profile)) {
    ProfilerRegisterThread();  // register for ITIMER_PROF signal
} else {
    ngx_log_error(NGX_LOG_CRIT, ...);  // log failure
}
```

`ProfilerStart` returns non-zero on success. `ProfilerRegisterThread` is required for signal-based CPU sampling.

#### Step E: cleanup

```
ngx_free(profile);  // free temporary filename buffer
return NGX_OK;
```

Note: profiler continues running — `ProfilerStop()` is called by profiler destructor on process exit.

### 3.3 Concrete walkthrough: `ngx_google_perftools_create_conf()` (line 73)

1. `ngx_pcalloc(cycle->pool, sizeof(ngx_google_perftools_conf_t))` — allocate zeroed config
2. `profiles` field is implicitly `{ 0, NULL }` from `pcalloc` — signals "not configured"
3. Returns config pointer to framework for storage in `cycle->conf_ctx`

---

## 4. Fast Mental Model for New Engineers

- **Two files, one real module**: only `ngx_google_perftools_module.c` runs at runtime; cpp test is build-only
- **Core module, process hook**: registered as `NGX_CORE_MODULE`, activates via `init_process` per worker
- **Per-worker isolation**: each worker writes its own profile file (`path.pid`)
- **Inherited state cleanup**: worker sanitizes master's profiler state before starting its own
- **No request-path cost**: profiling overhead is from `ITIMER_PROF` signals, not inline instrumentation

---

## 5. Recommended First Read Order in `src/misc`

1. `src/misc/ngx_google_perftools_module.c` — entire file (130 lines), complete module lifecycle
2. `src/misc/ngx_cpp_test_module.cpp` — skim for header inclusion pattern

---

## 6. Practical Debug Checklist

When debugging profiler behavior:

1. Is `google_perftools_profiles` configured in the `main` context? If not, module no-ops.
2. Is `-lprofiler` linked? Missing link dependency causes undefined symbol at load.
3. Is the output path writable by the worker user? `ProfilerStart` fails silently on permission errors.
4. Are profile files generated per worker PID? Check for `<prefix>.<pid>` files.
5. Is `CPUPROFILE` env var set? Could cause unexpected profiler state inheritance.
6. Is `ITIMER_PROF` conflicting with other signal-based tools (e.g., gprof)?
7. Is runtime overhead acceptable under peak traffic? Profile sampling uses periodic timer interrupts.
8. On process exit, is `ProfilerStop()` being called by the profiler destructor? Missing profiles may indicate abnormal termination.
