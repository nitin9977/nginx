# NGINX OS Module Ramp-Up Guide

**Audience**: Engineers onboarding to `src/os` internals  
**Scope**: Architecture, integration, and walkthrough for representative OS-path execution  
**Date**: 2026-05-19  
**Source paths**: All `src/` references are relative to the **nginx repository root** (`<repo>/src/`), not relative to `.github/`.

---

## 1. Architecture and Design of `src/os`

### 1.1 What `src/os` is

`src/os` is nginx's platform adaptation layer. It isolates OS-specific system calls, process control, socket behavior, and low-level runtime initialization.

Top-level split:

- `src/os/unix` — POSIX-centric process/event/signal/fs/net helpers
- `src/os/win32` — Windows-specific process and network extensions (AcceptEx, IOCP helpers, etc.)

### 1.2 Design goals

- Keep upper modules mostly platform-agnostic
- Normalize core runtime facts (`ngx_pagesize`, `ngx_ncpu`, `ngx_max_sockets`, I/O hooks)
- Encapsulate process/signal/daemon semantics per platform
- Expose capability flags used by core and event logic

### 1.3 Key design objects

#### `ngx_os_io_t` (`src/os/unix/ngx_os.h` / `src/os/unix/ngx_posix_init.c`)

I/O function table — wired into `ngx_connection_t` at accept time:

- `recv`: `ngx_unix_recv` — single-buffer `recv()`
- `recv_chain`: `ngx_readv_chain` — scatter/gather `readv()`
- `udp_recv`: `ngx_udp_unix_recv` — UDP `recvmsg()`
- `send`: `ngx_unix_send` — single-buffer `send()`
- `udp_send`: `ngx_udp_unix_send` — UDP `sendto()`
- `udp_sendmsg_chain`: `ngx_udp_unix_sendmsg_chain` — UDP `sendmsg()` chain
- `send_chain`: `ngx_writev_chain` — gather `writev()`, replaced by platform-specific `sendfile` variants
- `flags`: reserved (currently 0)

Platform-specific `sendfile` chain implementations override `send_chain`:

- `ngx_linux_sendfile_chain` — Linux `sendfile()` + `writev()` fallback
- `ngx_freebsd_sendfile_chain` — FreeBSD `sendfile()` with headers/trailers
- `ngx_darwin_sendfile_chain` — macOS `sendfile()` variant
- `ngx_solaris_sendfilev_chain` — Solaris `sendfilev()`

#### `ngx_process_t` (`src/os/unix/ngx_process.h`)

Per-process state in the global `ngx_processes[NGX_MAX_PROCESSES]` array:

- `pid`: process ID
- `status`: `waitpid()` exit status
- `channel[2]`: socketpair for master↔worker IPC (channel[0]=master, channel[1]=worker)
- `proc`: spawn callback (`ngx_spawn_proc_pt`) — the worker's main function
- `data`: opaque data passed to `proc`
- `name`: process name string (e.g., "worker process")
- `respawn:1`: process should be respawned on exit
- `just_spawn:1`: newly spawned (not yet signaled)
- `detached:1`: runs independently (e.g., hot-upgrade binary)
- `exiting:1`: shutdown in progress
- `exited:1`: process has exited

#### Signal flag variables (`src/os/unix/ngx_process_cycle.c`)

All declared as `sig_atomic_t` for async-signal safety:

- `ngx_reap` — SIGCHLD received, reap children
- `ngx_terminate` — fast shutdown (SIGTERM)
- `ngx_quit` — graceful shutdown (SIGQUIT)
- `ngx_reconfigure` — reload config (SIGHUP)
- `ngx_reopen` — reopen log files (SIGUSR1)
- `ngx_change_binary` — hot binary upgrade (SIGUSR2)
- `ngx_noaccept` — stop accepting (SIGWINCH)
- `ngx_sigalrm` — timer alarm for termination delay
- `ngx_sigio` — I/O signal counter

### 1.4 Unix directory file inventory

`src/os/unix` contains 65+ files organized by subsystem:

**Memory**: `ngx_alloc.c/.h` — `ngx_alloc()`/`ngx_calloc()`/`ngx_memalign()` wrappers around `malloc`/`posix_memalign`

**Process control**: `ngx_process.c/.h` — `ngx_spawn_process()`, `ngx_execute()`, `ngx_init_signals()`; `ngx_process_cycle.c/.h` — master/worker/single loops, signal dispatch

**IPC**: `ngx_channel.c/.h` — `ngx_write_channel()`/`ngx_read_channel()` over Unix domain socketpairs

**Daemon**: `ngx_daemon.c` — double-fork daemonization

**Platform init**: `ngx_posix_init.c` — common POSIX init; `ngx_linux_init.c`, `ngx_freebsd_init.c`, `ngx_darwin_init.c`, `ngx_solaris_init.c` — OS-specific discovery (e.g., Linux `sendfile` flags, FreeBSD `accept_filter`)

**I/O**: `ngx_recv.c`, `ngx_readv_chain.c`, `ngx_send.c`, `ngx_writev_chain.c`, `ngx_udp_recv.c`, `ngx_udp_send.c`, `ngx_udp_sendmsg_chain.c` — I/O function implementations

**Sendfile**: `ngx_linux_sendfile_chain.c`, `ngx_freebsd_sendfile_chain.c`, `ngx_darwin_sendfile_chain.c`, `ngx_solaris_sendfilev_chain.c`

**Files**: `ngx_files.c/.h` — file operations, directory listing, file info wrappers

**Shared memory**: `ngx_shmem.c/.h` — `mmap`/`shmget` wrappers

**Sockets**: `ngx_socket.c/.h` — `ngx_nonblocking()`, `ngx_blocking()`, TCP options

**Threading**: `ngx_thread.h`, `ngx_thread_mutex.c`, `ngx_thread_cond.c`, `ngx_thread_id.c`

**Atomics**: `ngx_atomic.h`, `ngx_gcc_atomic_*.h`, `ngx_sunpro_atomic_*.h` — platform-specific CAS/atomic ops

**Time**: `ngx_time.c/.h` — `gettimeofday()` wrapper, timezone helpers

**Affinity**: `ngx_setaffinity.c/.h` — CPU affinity binding

**Errno**: `ngx_errno.c/.h` — error string helpers

### 1.5 Unix model: `ngx_os_init()` (`src/os/unix/ngx_posix_init.c`, line 36)

Initialization sequence:

1. `ngx_os_specific_init()` — platform-specific probes (Linux: sendfile flags; FreeBSD: accept filter; Darwin: sendfile variant)
2. `ngx_init_setproctitle()` — prepare argv for process title changes
3. `ngx_pagesize = getpagesize()` and compute `ngx_pagesize_shift`
4. Set `ngx_cacheline_size` from `sysconf(_SC_LEVEL1_DCACHE_LINESIZE)` or compile-time default
5. `ngx_ncpu = sysconf(_SC_NPROCESSORS_ONLN)` — CPU count
6. `ngx_cpuinfo()` — x86 CPUID detection
7. `getrlimit(RLIMIT_NOFILE)` → `ngx_max_sockets`
8. Set `ngx_inherited_nonblocking` based on `accept4()` support
9. `srandom()` seeded from pid + time

### 1.6 Win32 model

`ngx_os_init()` in `src/os/win32/ngx_win32_init.c` initializes:

- Windows version/runtime capabilities
- Winsock startup (`WSAStartup`)
- Extension function pointers via `WSAIoctl` (AcceptEx, ConnectEx, TransmitFile, etc.)
- Processor/page/alignment metadata

Design implication: many async network primitives on Windows are discovered dynamically via `SIO_GET_EXTENSION_FUNCTION_POINTER`.

---

## 2. Integration and Dependencies

### 2.1 Integration with `src/core` startup

`main()` in core calls `ngx_os_init()` very early. Core subsystems assume OS init completed before:

- CRC32 table init (depends on `ngx_cacheline_size`)
- Slab allocator (depends on `ngx_pagesize`)
- Pool allocator (depends on `ngx_pagesize` for max threshold)
- Connection array sizing (depends on `ngx_max_sockets`)
- Signal setup (`ngx_init_signals()`)

### 2.2 Integration with `src/event`

Event backend behavior depends on OS-layer capabilities:

- `ngx_os_io` function table is copied into each connection's I/O pointers at accept
- Platform-specific sendfile chain replaces `send_chain` based on OS init
- Worker loops (`ngx_worker_process_cycle`) call `ngx_process_events_and_timers()`
- Signal masks in master loop control event loop interruption
- `ngx_inherited_nonblocking` affects whether `accept()`'d sockets need explicit `fcntl()`

### 2.3 Integration with process lifecycle

Master↔worker coordination uses:

- `ngx_spawn_process()` — `fork()`, sets up channel socketpair, stores in `ngx_processes[]`
- `ngx_signal_worker_processes()` — sends signals or channel messages to all workers
- `ngx_reap_children()` — `waitpid()` loop, handles respawn policy
- Channel commands: `NGX_CMD_QUIT`, `NGX_CMD_TERMINATE`, `NGX_CMD_REOPEN`, `NGX_CMD_OPEN_CHANNEL`, `NGX_CMD_CLOSE_CHANNEL`

### 2.4 External dependency integration

- POSIX syscalls on Unix (no external libraries beyond libc)
- Win32 API + Winsock + runtime-discovered extension functions on Windows
- Build-time feature macros from `auto/` configure probes define capability conditionals

---

## 3. Basic Code Walkthrough for OS Flows

### 3.1 Generic tracing template

1. Identify platform directory (`unix` or `win32`) used in this build
2. Locate OS init path and capability setup
3. Trace process-cycle loop and signal/control events
4. Confirm how event loop integration is invoked on that platform

### 3.2 Concrete walkthrough: unix startup to master/worker signal-driven loop

#### Step A: `ngx_os_init()` (posix_init.c line 36)

1. Platform-specific init: `ngx_os_specific_init()` (e.g., `ngx_linux_init()`)
2. `ngx_pagesize = getpagesize()` — typically 4096
3. `ngx_cacheline_size` from sysconf or compile-time default (64 bytes on x86)
4. `ngx_ncpu = sysconf(_SC_NPROCESSORS_ONLN)`
5. `ngx_max_sockets` from `RLIMIT_NOFILE`
6. Seed `srandom()` from pid + time

#### Step B: `ngx_master_process_cycle()` (process_cycle.c line 74)

1. Block signals: SIGCHLD, SIGALRM, SIGIO, SIGINT, plus nginx-specific signals (HUP, USR1, USR2, WINCH, TERM, QUIT)
2. Set process title to `"master process"` + argv
3. `ngx_start_worker_processes(cycle, ccf->worker_processes, NGX_PROCESS_RESPAWN)`
4. `ngx_start_cache_manager_processes(cycle, 0)`
5. Enter infinite loop:

#### Step C: master loop signal dispatch

Each iteration:

1. `sigsuspend(&set)` — atomically unblock signals and wait
2. `ngx_time_update()` — refresh cached time
3. Check signal flags in priority order:

| Flag | Signal | Action |
|------|--------|--------|
| `ngx_reap` | SIGCHLD | `ngx_reap_children()` — waitpid, respawn if needed |
| `ngx_terminate` | SIGTERM | Send TERM to workers, escalate to SIGKILL after delay |
| `ngx_quit` | SIGQUIT | Send QUIT to workers, close listening sockets |
| `ngx_reconfigure` | SIGHUP | `ngx_init_cycle()` → start new workers → signal old workers |
| `ngx_restart` | (internal) | Respawn all workers |
| `ngx_reopen` | SIGUSR1 | Reopen files, signal workers to reopen |
| `ngx_change_binary` | SIGUSR2 | `ngx_exec_new_binary()` — hot upgrade |
| `ngx_noaccept` | SIGWINCH | Signal workers to stop accepting |

4. If `!live && (ngx_terminate || ngx_quit)` → `ngx_master_process_exit()`

#### Step D: worker init — `ngx_worker_process_init()` (process_cycle.c)

1. Set environment variables
2. Set CPU affinity via `ngx_setaffinity()`
3. Set priority (`setpriority`), rlimits
4. Drop privileges: `setgid()`, `initgroups()`, `setuid()`
5. Close other workers' channel fds (but keep own `channel[1]`)
6. Call each module's `init_process` callback
7. Close listening sockets not in this worker's set
8. Register channel read event for master→worker commands

#### Step E: worker runtime loop

```
for (;;) {
    ngx_process_events_and_timers(cycle);
    if (ngx_terminate) { /* fast exit */ }
    if (ngx_quit) { /* graceful exit */ }
    if (ngx_reopen) { /* reopen log files */ }
}
```

### 3.3 Concrete walkthrough: reconfiguration (SIGHUP)

1. Master receives SIGHUP → `ngx_reconfigure = 1`
2. Loop iteration detects flag, calls `ngx_init_cycle(cycle)` — builds new cycle
3. If cycle creation fails, continues with old cycle
4. On success: starts new workers with `NGX_PROCESS_JUST_RESPAWN`, starts cache managers
5. `ngx_msleep(100)` — allow new workers to initialize
6. Signals old workers with `NGX_SHUTDOWN_SIGNAL` — graceful drain
7. Old workers finish existing connections, then exit

### 3.4 Alternate walkthrough: win32 network extension discovery

1. `ngx_os_init()` starts Winsock via `WSAStartup()`
2. Creates temporary dummy socket
3. `WSAIoctl(SIO_GET_EXTENSION_FUNCTION_POINTER)` obtains function pointers: AcceptEx, GetAcceptExSockaddrs, TransmitFile, TransmitPackets, ConnectEx, DisconnectEx
4. Event/network modules use these pointers for async I/O operations
5. Temporary socket closed

---

## 4. Fast Mental Model for New Engineers

- **OS layer = platform normalization**: upper layers never call `getpagesize()` or `accept4()` directly
- **`ngx_os_io` = I/O vtable**: recv/send function pointers wired into every connection
- **Signals are flags, not handlers**: signal handlers just set `sig_atomic_t` flags; the master loop dispatches
- **Master = signal dispatcher**: blocks with `sigsuspend()`, dispatches to worker management functions
- **Workers = event loops**: call `ngx_process_events_and_timers()` repeatedly, check signal flags per iteration
- **Channel = Unix socketpair**: master sends commands to workers when signals can't carry enough info
- **Process table is static**: `ngx_processes[1024]` is a fixed array, indexed by slot number

---

## 5. Recommended First Read Order in `src/os/unix`

1. `src/os/unix/ngx_process.h` — `ngx_process_t` struct and spawn API
2. `src/os/unix/ngx_os.h` — `ngx_os_io_t` function table
3. `src/os/unix/ngx_posix_init.c` — `ngx_os_init()` boot sequence
4. `src/os/unix/ngx_process_cycle.c` — master/worker loops and signal dispatch
5. `src/os/unix/ngx_process.c` — `ngx_spawn_process()`, `ngx_init_signals()`
6. `src/os/unix/ngx_channel.c` — IPC channel read/write
7. `src/os/unix/ngx_alloc.c` — low-level malloc wrappers
8. `src/os/unix/ngx_linux_init.c` (or your target platform) — platform-specific init

---

## 6. Practical Debug Checklist

When debugging OS-level behavior:

1. Did `ngx_os_init()` complete successfully? Check for ALERT-level log messages from pagesize/rlimit/cpuinfo probes.
2. Is `ngx_max_sockets` adequate? If `RLIMIT_NOFILE` is too low, connection pool will be undersized.
3. Is `ngx_cacheline_size` correct? Wrong value causes slab/shared-memory misalignment (silent performance degradation).
4. Did `ngx_spawn_process()` create the channel socketpair? Check for `socketpair()` failures.
5. Are signals being delivered? Check signal mask — master blocks signals then unblocks in `sigsuspend()`.
6. Is the master loop actually reaching `sigsuspend()`? A busy loop in signal dispatch can starve the suspend call.
7. Is worker respawn working? Check `ngx_reap_children()` — respawn policy depends on process flags.
8. Are channel commands received? Check `ngx_channel_handler()` read event on worker side.
9. Did privilege drop succeed? Check `setuid()`/`setgid()` return codes — failure may leave worker running as root.
10. On reconfigure, did `ngx_init_cycle()` succeed? Failed reload continues with old cycle but logs the error.
