# nginx PR Review System

**Status**: ✅ Fully Configured  
**Scope**: Memory safety, concurrency, performance, security  
**Mode**: Automatic (on PR) + Manual (on demand)  

---

## 📁 System Files

### Agents (`.github/agents/`)
- **`pr-reviewer.agent.md`** – Main PR review agent (memory, concurrency, security)
  - Code-review specialized agent type
  - High-signal, low-noise review methodology
  - Supports strict/focused/quick modes
  - Issues rated by severity (critical/high/medium/low)

- **`principal-engineer.agent.md`** – Architecture & systems design
  - Architectural validation
  - Threat modeling
  - Performance analysis
  - Adversarial assumption validation

### Hooks (`.github/hooks/`)
- **`pr-review.json`** – GitHub Actions automation
  - Triggers: PR open, PR synchronize, comment commands
  - Auto-runs in `focused` mode (critical + high issues)
  - Parses `/review strict|focused|quick` commands
  - Posts results as GitHub PR comment

### Instructions (`.github/instructions/`)
- **`nginx-c-review.instructions.md`** – C code review guidelines
  - Applies to: `src/**/*.{c,h}`, `auto/**/*.c`, `conf/**/*.c`
  - Memory safety checklist
  - Concurrency validation checklist
  - Correctness & logic checklist
  - Performance checklist
  - Red flags to watch for
  - Example issues (what to flag vs skip)

### Documentation (`.github/`)
- **`AGENTS.md`** – Registry of custom agents
- **`AGENTS.md`** – How to invoke agents
- **`PR_REVIEW_GUIDE.md`** – Complete user guide for review system
- **`REVIEW_SYSTEM.md`** – This file

---

## 🚀 Quick Start

### Automatic Review (No action needed)
1. Open PR in GitHub
2. Review agent automatically analyzes changes
3. Comment posted with findings

### Manual Review (In VS Code)
```
@pr-reviewer: Review this change for memory safety
@pr-reviewer: Analyze performance impact
/review strict      # Get all issues
/review focused     # Get critical + high (default)
/review quick       # Get critical only
```

### Architecture Review (For design questions)
```
@principal-engineer: Validate this new concurrency model
@principal-engineer: Threat model the connection handling changes
```

---

## 🎯 What Gets Reviewed

✅ **Reviewed**:
- Buffer bounds and overflow risk
- Memory allocation/deallocation
- Use-after-free and double-free
- Race conditions and synchronization
- Lock ordering violations
- Unvalidated input
- Performance regressions
- Error handling
- Signal safety

❌ **Not Reviewed** (handled by linters):
- Code style and formatting
- Naming conventions
- Comment style
- Whitespace
- Pre-existing issues in unchanged code

---

## 📊 Review Severity Levels

| Level | Action | Examples |
|-------|--------|----------|
| 🔴 **Critical** | Blocks merge | Buffer overflow, use-after-free, race condition, security vulnerability |
| 🟠 **High** | Strongly recommend fix | Unvalidated input, missing error check, lock ordering issue |
| 🟡 **Medium** | Recommend fix | Incomplete bounds checking, inefficient pattern, config risk |
| 🟢 **Low** | Optional | Simplification opportunity, documentation gap |

---

## 🔄 Integration Points

### With Code Linting
- Linters: Catch style, naming, basic correctness
- Review agents: Catch memory safety, concurrency, performance

### With Code Coverage
- Coverage tools: Measure line coverage
- Review agents: Validate critical paths and edge cases

### With Static Analysis
- clang-analyzer: Find obvious bugs
- Review agents: Understand implications and propose fixes

### With Testing
- Unit tests: Validate behavior
- Review agents: Suggest test scenarios and edge cases

---

## 🛠️ Configuration Files

### `.github/hooks/pr-review.json`
Controls automatic review triggering:
```json
{
  "event": "pull_request",
  "triggers": ["opened", "synchronize"],
  "mode": "focused",
  "paths": ["src/**", "conf/**", "auto/**"],
  "exclude_paths": ["docs/**", "contrib/**"]
}
```

**To change:**
- Add/remove path patterns
- Change default mode (strict/focused/quick)
- Add/remove trigger types

### `.github/instructions/nginx-c-review.instructions.md`
Defines review scope for C files. **To update:**
- Add new red flags
- Add new checklists
- Update tool commands
- Add example patterns

---

## 📝 Review Output Format

All reviews follow this structure:

```markdown
## 🔍 Automated PR Review (@pr-reviewer)

### Summary
[1-2 sentence overview]

### 🔴 Critical Issues
#### Issue 1: [Category]
**Location**: `file.c:42`
**Problem**: [Description with code reference]
**Impact**: [Why this matters]
**Fix**: [Suggested code]

### 🟠 High Issues
[Same format]

### 🟡 Medium Issues
[Same format]

### ✅ Positive Notes
- [What the PR does well]

### 🧪 Test Coverage
[Assessment of test adequacy]

### ⚠️ Assumptions & Validation
[Non-obvious assumptions that should be validated]

### 📋 Recommended Actions
- [ ] Address critical issues
- [ ] Address high issues
- [ ] Run [specific tests]
```

---

## 🔧 Troubleshooting

### Review not triggering on PR
- Check GitHub Actions are enabled
- Verify `.github/hooks/pr-review.json` syntax
- Check PR changed files match `paths` patterns
- Look at GitHub Actions logs

### Review comment not appearing
- Check if PR changed `src/`, `auto/`, or `conf/` directories
- Verify PR is not draft (if `skip_if_draft: true`)
- Check GitHub Actions execution status

### Getting different results locally vs automatic
- Same review mode? (strict vs focused vs quick)
- Same agent? (@pr-reviewer vs @principal-engineer)
- Different diff? (newer commits might change findings)

### Agent not found
- Check agent files are in `.github/agents/`
- Verify YAML frontmatter syntax
- Reload VS Code chat context

---

## 📚 See Also

- **User Guide**: `.github/PR_REVIEW_GUIDE.md`
- **Agent Registry**: `.github/AGENTS.md`
- **C Code Review Guidelines**: `.github/instructions/nginx-c-review.instructions.md`
- **PR Reviewer Details**: `.github/agents/pr-reviewer.agent.md`
- **Principal Engineer Details**: `.github/agents/principal-engineer.agent.md`
- **Architecture Docs**: `.github/docs/architecture/`

---

**System created**: 2026-05-13  
**Status**: ✅ Ready for use  
**Last reviewed**: 2026-05-13
