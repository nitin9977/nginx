# nginx PR Review Guide

This guide explains how to use the automated and manual PR review systems for the nginx repository.

## Overview

The nginx repository has two specialized review agents working together:

| Agent | Scope | Trigger | Use Case |
|-------|-------|---------|----------|
| **@principal-engineer** | Architecture & systems design | Manual (chat) | Architectural changes, design reviews, threat models |
| **@pr-reviewer** | Code correctness & safety | Auto + manual | Day-to-day PR reviews, memory/concurrency validation |

---

## 🤖 Automated PR Reviews

**What happens:**
- Every PR automatically triggers a code review when opened or updated
- Review runs in `focused` mode (critical + high severity issues only)
- Results posted as a GitHub comment on the PR

**What's reviewed:** Changes under `src/`, `auto/`, `conf/` (`.c`/`.h`) — see the review categories and checks in [pr-reviewer.agent.md](.github/agents/pr-reviewer.agent.md#review-categories). Skips docs, contrib, and style/formatting (linter territory).

**How to control it:**

After the automatic review is posted, you can request a different verbosity level by commenting on the PR:

```bash
# Get all issues (strict mode)
/review strict

# Get only critical issues (quick mode)
/review quick

# Back to default (critical + high)
/review focused
```

**Example flow:**
1. Engineer opens PR with code changes
2. GitHub Action auto-triggers `@pr-reviewer` in `focused` mode
3. Comment posted with findings (critical + high severity)
4. Engineer reviews, fixes issues, pushes update
5. Review auto-updates
6. Engineer comments `/review strict` if they want complete analysis
7. Updated review shows all issues (critical, high, medium, low)

---

## 👤 Manual PR Reviews

### When to use @pr-reviewer manually:

**In VS Code chat:**
```
@pr-reviewer: Review src/core/ngx_connection.c for memory safety issues

@pr-reviewer: Deep analysis of this performance-critical change
/review strict
```

**Best for:**
- Complex or risky changes (networking, event loop, memory management)
- Before you push to ensure quality
- Getting a second opinion on tricky logic
- Validating assumptions before code review discussion
- Deep-diving into security implications

### Review Modes:

| Mode | When to use | Output |
|------|-------------|--------|
| **strict** | Complex/risky changes, security-sensitive code | All issues (critical, high, medium, low) + full analysis |
| **focused** | Standard PRs, day-to-day changes | Critical + high issues only (default) |
| **quick** | Quick sanity check | Critical issues only, brief explanation |
| *(default)* | Most PRs | Critical + high + medium issues |

---

## 🏗️ Escalating to Architecture Review

If a PR involves **architectural decisions** (not just code fixes):

```
@principal-engineer: Review this PR for architectural impact
- Explain the new worker pool design
- Validate assumptions about concurrency
- Threat-model the new connection lifecycle

Context: This PR introduces a new event handling strategy in ngx_event.c
```

**When to escalate:**
- New subsystems or major modules
- Concurrency model changes
- Memory allocation strategy redesigns
- Event loop modifications
- Significant performance optimizations

**What to expect:**
- Multi-part analysis (architecture, memory model, threat model)
- Code walkthroughs with invariants
- Validation procedures and assumptions testing
- Risk assessment (critical/high/medium/low)

---

## 📋 Review Checklist for Authors

Before requesting review:

- [ ] Code compiles with `-Wall -Wextra -Wpedantic`
- [ ] Passes ASAN/UBSAN: `clang -fsanitize=address,undefined`
- [ ] For concurrency changes: passes ThreadSanitizer: `clang -fsanitize=thread`
- [ ] New code has tests
- [ ] Edge cases tested (0, max value, empty input)
- [ ] Error paths handled
- [ ] No compiler warnings
- [ ] Consistent with nginx coding style (run `clang-format`)

---

## 🚨 What Gets Flagged

Severity definitions (critical/high/medium/low) and the full "what's flagged vs. skipped" breakdown live in [pr-reviewer.agent.md](.github/agents/pr-reviewer.agent.md#review-categories) — that file is the source of truth, kept in sync with the review output.

---

## 🔗 Integration with GitHub

Trigger events, path filters, default mode, and the `/review` comment-command parsing are all defined in [.github/hooks/pr-review.json](.github/hooks/pr-review.json) — edit that file directly to change automation behavior rather than duplicating its settings here.

---

## 📖 Reference: Knowledge Base

All review criteria (memory safety, concurrency, correctness, performance, nginx-specific conventions) and the domain knowledge behind them (module ownership, struct layouts, HTTP RFC semantics, directive contracts) live in [.github/docs/](.github/docs/) and [.github/skills/](.github/skills/) — see the Knowledge Base section of [pr-reviewer.agent.md](.github/agents/pr-reviewer.agent.md#knowledge-base) for the full list.

---

## 🎯 Example: PR Review Flow

### Scenario: Performance optimization PR

**Step 1: Author opens PR**
```
Title: "Optimize connection accept loop for high throughput"
Description: Reduce lock contention in accept loop by batching...
```

**Step 2: Automatic review triggered**
- GitHub Action runs `@pr-reviewer` in `focused` mode
- Comment posted with critical + high issues found

**Step 3: Author reviews findings**
- Sees 2 critical issues (potential race condition, buffer sizing)
- Sees 3 high issues (lock ordering, missing error check)
- Agrees these need fixing

**Step 4: Author can request more detail**
```
/review strict
```
- Updated comment now shows medium + low issues
- Includes optimization suggestions
- References specific line numbers and functions

**Step 5: Author fixes and pushes**
- Commits fix for race condition
- Reviews auto-updates with new diff
- No more critical issues found

**Step 6: For architectural validation**
```
@principal-engineer: The accept loop optimization changes our lock strategy.
Please validate the new concurrency model against the threat model.
```
- Principal Engineer provides deep analysis
- Validates assumptions about contention patterns
- Suggests benchmarking procedures

---

## 🤝 Best Practices

### For Authors:
1. **Run local checks first**: ASAN, clang-format before pushing
2. **Start with quick review**: `@pr-reviewer` locally if uncertain
3. **Respond to findings**: Fix critical issues before requesting code review
4. **Escalate thoughtfully**: Use `@principal-engineer` for architectural questions
5. **Test edge cases**: Have tests for the scenarios reviewers might ask about

### For Reviewers:
1. **Check automated review**: See what `@pr-reviewer` found first
2. **Focus on high-level logic**: Not style (automated checks cover that)
3. **Look for assumptions**: Ask "why?" not just "what?"
4. **Validate concurrency**: Never assume locks are sufficient without verification
5. **Escalate risks**: If security/performance concern, mention it explicitly

### For Team:
1. **Trust but verify**: Automated review is a tool, not gospel (use judgment)
2. **Document decisions**: If you override a review finding, document why
3. **Improve checklists**: Update guidelines based on issues found
4. **Validate assumptions**: Use threat models and tests to verify claims

---

## 📞 Support

For questions about the review system:
- Check [pr-reviewer.agent.md](.github/agents/pr-reviewer.agent.md) for detailed review methodology and its knowledge base
- Review [.github/hooks/pr-review.json](.github/hooks/pr-review.json) for automation configuration
- Ask `@principal-engineer` for architectural guidance

For setup or troubleshooting:
- Verify files are in `.github/` directory
- Check GitHub Actions logs for hook execution
- Run `@pr-reviewer` manually in VS Code to test

---

**Last updated**: 2026-05-13  
**Maintained by**: Principal Engineer Agent
