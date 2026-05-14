---
name: nginx-c-review-guidelines
description: >
  Review guidelines for C source files in nginx. Applies to all .c and .h files
  in src/, auto/, and conf/ directories. Focuses on memory safety, concurrency,
  and performance issues critical to a systems-level HTTP server.
applyTo: ["src/**/*.{c,h}", "auto/**/*.c", "conf/**/*.c"]
---

# nginx C Code Review Guidelines

## Critical Focus Areas

When reviewing C code changes in nginx, prioritize these checks in order:

### 1. Memory Safety (🔴 if violated)
- **Buffer bounds**: Are all buffer accesses bounds-checked?
  - Array indexing within bounds?
  - String operations (strcpy, strcat, sprintf) use safe variants?
  - Integer overflow possible in size calculations?
- **Lifetime correctness**: Is allocated memory freed?
  - No use-after-free?
  - No double-free?
  - Lifetime matches scope?
- **Pointer validity**: Are pointers checked before use?
  - NULL checks after allocation?
  - Dangling pointers avoided?
  - Memory not accessed after free?

**Validation**: Run with AddressSanitizer (ASAN) and Valgrind

### 2. Concurrency (🔴 if violated)
- **Race conditions**: Is shared state properly synchronized?
  - All accesses to shared data protected by locks?
  - Lock ordering consistent? (same order everywhere)
  - Memory barriers placed correctly?
  - Atomic operations used where appropriate?
- **Deadlock prevention**: Could locks deadlock?
  - Do we acquire locks in same order across threads?
  - Any nested locks without documented ordering?
  - Timeout or non-blocking used where appropriate?
- **Signal safety**: Are signal handlers async-signal-safe?
  - Only calling async-signal-safe functions?
  - No access to signal-unsafe state?

**Validation**: ThreadSanitizer (TSAN), code inspection for lock ordering

### 3. Correctness & Logic (🔴 if violated)
- **Error handling**: Are errors propagated correctly?
  - Syscall return values checked?
  - Error codes propagated up the stack?
  - No silent failures?
- **Edge cases**: Do code paths handle boundary conditions?
  - Empty inputs?
  - Maximum values (INT_MAX, 0xFFFFFFFF)?
  - Resource exhaustion?
  - Concurrent modifications?
- **Invariants**: Are preconditions/postconditions maintained?
  - State machines in correct states?
  - Protocol invariants maintained?

**Validation**: Unit tests for edge cases, fuzz testing

### 4. Performance (🟠 if violated)
- **Algorithmic complexity**: Any unexpected O(n²) or worse?
  - Nested loops over large datasets?
  - Linear searches where index lookups possible?
- **Resource efficiency**: Are resources allocated efficiently?
  - Excessive allocations in hot paths?
  - Unnecessary copies in data movement?
  - Cache locality considered?
- **Lock contention**: Will locks serialize critical sections?
  - Hold time of locks acceptable?
  - Lock-free alternatives viable?

**Validation**: Profiling, benchmarking, code review

### 5. Portability & Compatibility (🟡 if violated)
- **Platform assumptions**: What OS/architecture is assumed?
  - POSIX compliance?
  - Endianness assumptions?
  - Integer size assumptions (int vs long vs int64_t)?
- **Standard compliance**: Do we use standard C correctly?
  - Undefined behavior avoided?
  - Implementation-defined behavior documented?

**Validation**: Cross-platform testing, static analysis (clang-analyzer)

---

## Code Review Checklist

For each C file change, verify:

### Memory
- [ ] All allocations have matching deallocations
- [ ] Buffer sizes calculated without overflow
- [ ] String operations use bounds-safe variants
- [ ] Pointers checked for NULL before use
- [ ] No arrays stack-allocated with unbounded sizes

### Concurrency
- [ ] Shared state identified and protected
- [ ] Lock ordering documented and consistent
- [ ] Signal handlers only call async-signal-safe functions
- [ ] Atomic operations used for non-lockable data
- [ ] Memory barriers placed after lock release

### Correctness
- [ ] All error codes checked and handled
- [ ] Edge cases tested (0, 1, max_value, OOM)
- [ ] Protocol/state machine invariants maintained
- [ ] No silent failures on error

### Performance
- [ ] Hot paths identified
- [ ] No O(n²) algorithms in hot paths
- [ ] Allocations not in tight loops
- [ ] Cache-friendly data structures used where feasible

### Testing
- [ ] New code has test coverage
- [ ] Edge cases tested
- [ ] Error paths tested
- [ ] Concurrency tested (if applicable)

---

## Red Flags 🚩

Stop and ask for clarification if you see:

- **Void pointer arithmetic** — `(void*)ptr + 1` is undefined behavior
- **Unaligned memory access** — Can cause crashes on ARM/MIPS
- **Manual memory management without RAII pattern** — High error risk
- **Locks held across blocking operations** — Deadlock risk
- **Complex bit manipulation without comments** — Maintenance risk
- **Global mutable state without synchronization** — Race condition risk
- **Fixed-size buffers with unbounded input** — Overflow risk
- **Recursive syscalls** — May not be async-signal-safe

---

## Style Issues to Ignore

✅ **DO NOT flag these** — linters and formatters handle them:
- Indentation, spacing, brace style
- Variable naming (unless truly cryptic)
- Line length (unless >120 chars)
- Comment style
- Header guard format

---

## Reference Tools

When reviewing, have these available:

```bash
# Memory safety
clang -fsanitize=address,undefined -g
valgrind --leak-check=full

# Concurrency
clang -fsanitize=thread

# Static analysis
clang --analyze

# Code style (for reference, not to block)
clang-format --style=file
```

---

## Example: What to Flag vs What to Skip

### ❌ DON'T flag:
```c
// Inconsistent spacing around operators (formatter's job)
int x=a+b;  // vs int x = a + b;

// Naming style (unless truly cryptic)
int tmp = calculate();  // vs int result = calculate();

// Brace style
if (x) {    // vs if (x)
    ...           {
}                     ...
                  }
```

### ✅ DO flag:
```c
// Buffer overflow risk
char buf[64];
strcpy(buf, user_input);  // ← NO bounds check!

// Use-after-free
free(ptr);
printf("%d\n", ptr->value);  // ← Accessing freed memory!

// Race condition
shared_var++;  // ← Unsynchronized access to shared state!

// Integer overflow
size_t size = num_items * item_size;  // ← Overflow possible!
```

---

## Questions for Authors

When you see concerning patterns, ask:

1. **Memory**: "Can this buffer overflow? What's the maximum input size?"
2. **Concurrency**: "Which data is shared between threads? How is it protected?"
3. **Error handling**: "What happens if this syscall fails? Is the error propagated?"
4. **Performance**: "Is this in a hot path? Could it be optimized?"
5. **Assumptions**: "What are we assuming about the OS/hardware here?"

---

## Escalation to Principal Engineer

If the PR involves:
- **Architectural changes** (new subsystem, major refactor)
- **Concurrency redesign** (new locking strategy, lock-free changes)
- **Performance optimization** (new algorithms, memory management strategy)

Consider escalating to `@principal-engineer` for deeper review.
