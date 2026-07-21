# Custom Agents for nginx Repository

This directory contains specialized agents configured for the nginx project.

## Workspace Layout

Agents that cross-reference the `nginx-tests` suite (currently `@pr-reviewer`)
locate both repos in this order:

1. **Explicit env vars** — always win, use these when your checkouts aren't
   siblings:
   ```sh
   export NGINX_REPO_PATH=~/code/nginx
   export NGINX_TESTS_REPO_PATH=~/code/nginx-tests
   ```
2. **Sibling layout** — no configuration needed:
   ```
   ~/code/
   ├── nginx/            # fork of nginx/nginx
   └── nginx-tests/      # fork of nginx/nginx-tests
   ```
3. **Interactive fallback** — if a cross-repo check is required and neither of
   the above resolves, the agent will ask once for the missing path.

Single-repo reviews (nginx-only, or nginx-tests-only that doesn't need the C
sources) don't require the other repo to be locatable.

## Available Agents

### `@principal-engineer` – Architecture, Debugging & Feature Design Lead

**Use when:**
- Debugging nginx issues (crashes, memory leaks, connection hangs, config reload failures, signal problems, event loop stalls, phase engine bugs, upstream failures)
- Designing new nginx modules, features, phase handlers, or filters
- Generating architecture documentation for team onboarding
- Validating assumptions about performance, memory, or concurrency
- Threat-modeling critical subsystems
- Creating design specs or annotated code walkthroughs

**Knowledge base** (loaded automatically — no codebase traversal needed):
- `nginx-internals` skill with 5 reference files: module map, struct reference, lifecycle flows, debug playbook, design patterns
- `http-rfc-standards` skill: RFC 9110–9114 (HTTP semantics, caching, /1.1, /2, /3), RFC 9931 (smuggling defenses), and a compatibility map to the obsoleted 723x/2616/2068/1945 RFCs
- nginx Coding Style Guide (`.github/docs/coding-style/`) — mandatory formatting/naming/structural rules applied to every design output
- 14 architecture docs in `.github/docs/` covering all `src/` modules
- Directive reference (`.github/docs/directives/`) — full nginx.org directive index (core, HTTP, stream, mail) for config-correct designs and debugging

**Specialization:**
- C/C++ memory management and buffer safety
- High-performance networking architecture
- Concurrency models (workers, event loops, locking)
- Performance assumptions and bottleneck analysis
- System-level threat modeling
- Root cause analysis with structured debug workflows

**Engagement model:**
- For debugging: classifies symptom → consults debug playbook → traces execution path → reads only targeted source → provides root cause + fix
- For feature design: identifies pattern → loads design templates → designs with invariants → produces implementation skeleton
- For architecture: challenges assumptions, identifies risks, suggests hardening
- References specific line numbers and code locations
- Rates risks (critical/high/medium/low) and suggests validation tests

**Example usage:**
```
@principal-engineer: worker segfaults after config reload — trace the issue
@principal-engineer: design a stream module that rate-limits by source IP
@principal-engineer: Create architecture ramp-up docs for the event loop
@principal-engineer: Threat model: connection pooling under slowloris attack
@principal-engineer: is this proxy response handling vulnerable to request smuggling? (checks RFC 9112 §11.2)
```

---

### `@pr-reviewer` – Automated PR Code Review

**Use when:**
- Reviewing PRs for memory safety and correctness
- Validating security and concurrency in changes
- Deep-diving into performance impact of code changes
- Getting a second opinion before merge
- Triggering automated review on PR (automatic + manual)

**Knowledge base** (loaded automatically — no codebase traversal needed):
- nginx Coding Style Guide (`.github/docs/coding-style/`) — authoritative source for structural/naming checks (function ordering, type suffixes, directive conventions) that clang-format can't catch
- `nginx-internals` skill (module map, struct reference, design patterns) — identifies the owning module and its established conventions for the changed file
- `http-rfc-standards` skill: RFC 9110–9114, 9931 — cited when the diff touches HTTP parsing, framing, caching, or h2/h3 code, especially for request-smuggling risk
- Directive reference (`.github/docs/directives/`) — verifies documented defaults/context/syntax when a diff adds or changes a config directive

**Specialization:**
- Buffer overflow and memory safety vulnerabilities
- Race conditions and concurrency issues
- Performance regressions and bottleneck analysis
- Security vulnerabilities (DoS, injection, etc.)
- Correctness and logic errors
- Cross-platform compatibility

**Engagement model:**
- **High-signal, low-noise**: Only flags genuine issues (bugs, security, memory/concurrency)
- **Never nitpicks style**: Formatting, naming, comments are for linters
- **Always actionable**: Suggests concrete fixes with code examples
- **Configurable verbosity**: `strict` (all issues), `focused` (critical + high), `quick` (critical only)

**Automatic triggers:**
- On PR open and push: Runs in `focused` mode
- On comment: `/review strict` or `/review quick` for manual control

**Example usage:**
```
@pr-reviewer: Review this change for concurrency issues

/review strict          # All issues, deep analysis
/review focused         # Critical + high only (default)
/review quick           # Critical issues only
```

**Integration:**
- Works alongside `@principal-engineer`
- PR Reviewer validates individual changes (low-level)
- Principal Engineer validates architecture (high-level)
- Escalate architectural concerns with `@principal-engineer`

---

## Invocation

### Manual (In VS Code chat):
- Type `@pr-reviewer` followed by your review request
- Or use the command palette to invoke directly

### Automatic (GitHub):
- PR review runs automatically on PR open/update in `focused` mode
- Override with comment: `/review strict`, `/review focused`, or `/review quick`
- Review comments are posted as GitHub PR comments (thread-based)

## How It Works

**Hook Configuration** (`.github/hooks/pr-review.json`):
- Triggers automatic review on PR events
- Parses `/review` commands in comments
- Posts results as GitHub comments
- Filters to code paths (ignores docs, contrib)

**Agent** (`.github/agents/pr-reviewer.agent.md`):
- Handles detailed code analysis
- Produces structured review output
- Supports multiple verbosity modes

## Documentation Standards

When agents produce documentation, save to:
- `.github/docs/architecture/` – Design specs, diagrams, threat models
- `.github/docs/onboarding/` – Ramp-up guides, code walkthroughs
- `.github/docs/performance/` – Memory models, bottleneck analysis

All docs should follow:
- **Clarity**: Readable to engineers 1-5 years into their careers
- **Specificity**: Reference code locations, not vague concepts
- **Risk clarity**: Flag assumptions that could break the system
- **Validation**: Attach test strategies or verification methods
- **Honesty**: Acknowledge unknowns and constraints
