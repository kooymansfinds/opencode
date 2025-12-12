# OpenCode Test Suite Analysis

**Date:** 2025-12-12
**Test Location:** `packages/opencode/test/`
**Test Framework:** Bun's built-in test runner

---

## Overview

The OpenCode test suite provides comprehensive coverage for core functionality including:
- Bun runtime integration
- Configuration management
- File snapshot/tracking system
- Tool execution (Bash sandboxing)
- Markdown parsing for file references

**Total Test Files:** 5 main test suites
**Total Test Cases:** 50+ individual tests

---

## Test Suites

### 1. Bun Runtime Tests (`bun.test.ts`)

**Purpose:** Ensures Bun package manager integration doesn't use hardcoded registry configurations.

**Location:** `packages/opencode/test/bun.test.ts`

**What It Tests:**
- ✓ Verifies no hardcoded `--registry=` parameters
- ✓ Confirms use of Bun's default registry resolution
- ✓ Validates command structure for `bun add` operations
- ✓ Ensures proper handling of `.npmrc` and `bunfig.toml` files

**Key Test Cases:**
```typescript
test("should not contain hardcoded registry parameters")
test("should use Bun's default registry resolution")
test("should have correct command structure without registry")
```

**Why It Matters:**
This prevents the exact issue we encountered - hardcoded registry settings that bypass environment-specific proxy configurations. The tests ensure OpenCode respects user's npm/bun registry settings.

**Related File:** `packages/opencode/src/bun/index.ts` (the implementation being tested)

---

### 2. Configuration Tests (`config/config.test.ts`)

**Purpose:** Validates the configuration loading, merging, and validation system.

**Location:** `packages/opencode/test/config/config.test.ts`

**What It Tests:**

#### File Format Support
- ✓ JSON config loading (`opencode.json`)
- ✓ JSONC config loading with comments (`opencode.jsonc`)
- ✓ Default values when no config exists

#### Configuration Merging
- ✓ Multiple config file precedence (`opencode.json` > `opencode.jsonc`)
- ✓ Merging behavior across files
- ✓ Global vs project-level configs

#### Variable Substitution
- ✓ Environment variable interpolation: `{env:VAR_NAME}`
- ✓ File content inclusion: `{file:path/to/file}`
- ✓ Proper handling when variables are missing

#### Schema Validation
- ✓ Invalid field detection
- ✓ JSON parsing error handling
- ✓ Type checking enforcement

#### Configuration Features
- ✓ Agent/mode configuration
- ✓ Custom command definitions
- ✓ `.opencode/agent/` markdown file loading
- ✓ Legacy field migration (`mode` → `agent`, `autoshare` → `share`)

**Key Test Cases:**
```typescript
test("loads config with defaults when no files exist")
test("merges multiple config files with correct precedence")
test("handles environment variable substitution")
test("validates config schema and throws on invalid fields")
test("loads config from .opencode directory")
```

**Why It Matters:**
Configuration is the foundation of OpenCode's flexibility. These tests ensure users can:
- Customize behavior per project
- Use environment-specific settings securely
- Define custom agents and commands
- Migrate from older config formats seamlessly

**Related Files:**
- `packages/opencode/src/config/config.ts` - Configuration loader
- `packages/opencode/src/project/instance.ts` - Project context

---

### 3. Markdown File Reference Tests (`config/markdown.test.ts`)

**Purpose:** Tests the parser that extracts file references from markdown text using `@file/path` syntax.

**Location:** `packages/opencode/test/config/markdown.test.ts`

**What It Tests:**

#### File Reference Patterns
- ✓ Relative paths: `@valid/path/to/file`
- ✓ Absolute paths: `@/absolute/paths.txt`
- ✓ Home directory paths: `@~/home-files`
- ✓ Files with extensions: `@file.md`, `@multiple.extensions.bak`
- ✓ Hidden files: `@.bashrc`, `@.config/`

#### Edge Cases
- ✓ Files followed by punctuation: `@file.md,` or `@file.md.`
- ✓ Ignores backtick-quoted references: `` `@should/not/match` ``
- ✓ Distinguishes from email addresses: `user@example.com`

**Test Data:**
Uses a comprehensive template with 12 different file reference patterns to ensure robust parsing.

**Key Test Cases:**
```typescript
test("should extract exactly 12 file references")
test("should not match when preceded by backtick")
test("should not match email addresses")
```

**Why It Matters:**
This enables OpenCode's instruction system to reference specific files in custom prompts and agent configurations. Users can write documentation like:

```markdown
Review the authentication logic in @src/auth/index.ts and
ensure it follows patterns from @SECURITY.md.
```

**Related Files:**
- `packages/opencode/src/config/markdown.ts` - Markdown parser implementation

---

### 4. Snapshot/Tracking Tests (`snapshot/snapshot.test.ts`)

**Purpose:** Tests the git-based file change tracking and rollback system.

**Location:** `packages/opencode/test/snapshot/snapshot.test.ts`

**What It Tests:**

#### Core Snapshot Operations
- ✓ `Snapshot.track()` - Create snapshot hash of current state
- ✓ `Snapshot.patch()` - Generate list of changed files since snapshot
- ✓ `Snapshot.revert()` - Rollback changes to previous snapshot
- ✓ `Snapshot.diff()` - Show textual diff of changes
- ✓ `Snapshot.restore()` - Restore tracked files to snapshot state

#### File Operations Tracked
- ✓ New file creation
- ✓ File deletion
- ✓ File modification
- ✓ Directory creation (nested)
- ✓ Symlink creation and handling

#### Edge Cases & Special Files
- ✓ Binary files (images, etc.)
- ✓ Large files (1MB+)
- ✓ Unicode filenames (文件.txt, 🚀rocket.txt, café.txt)
- ✓ Files with spaces and special characters
- ✓ Very long filenames (200+ characters)
- ✓ Hidden files (`.gitignore`, `.config`)
- ✓ Circular symlinks (should not crash)
- ✓ Nested symlinks
- ✓ Files ignored by `.gitignore`

#### Correctness & Safety
- ✓ Empty patches (no changes) don't crash
- ✓ Invalid hashes handled gracefully
- ✓ Reverting non-existent files doesn't crash
- ✓ Concurrent file operations during snapshot
- ✓ State isolation between different projects
- ✓ Permission changes (not tracked, as expected)
- ✓ Multiple simultaneous file operations

**Key Test Cases (30+ tests):**
```typescript
test("tracks deleted files correctly")
test("revert should remove new files")
test("multiple file operations")
test("unicode filenames")
test("gitignore changes")
test("snapshot state isolation between projects")
test("track with no changes returns same hash")
```

**Implementation Details:**
- Uses Git under the hood for reliable change tracking
- Creates temporary git repos for each test (`tmpdir({ git: true })`)
- Leverages git's robust handling of edge cases

**Why It Matters:**
This is critical for AI agent safety - allows OpenCode to:
- Track all changes made by an AI agent
- Rollback problematic changes automatically
- Show users exactly what was modified
- Prevent accidental data loss
- Enable "undo" functionality for AI operations

**Related Files:**
- `packages/opencode/src/snapshot/index.ts` - Snapshot implementation
- Git integration for tracking

---

### 5. Bash Tool Tests (`tool/bash.test.ts`)

**Purpose:** Tests the sandboxed bash command execution tool.

**Location:** `packages/opencode/test/tool/bash.test.ts`

**What It Tests:**

#### Basic Execution
- ✓ Command execution returns proper exit codes
- ✓ Command output captured correctly
- ✓ Simple commands like `echo` work as expected

#### Security & Sandboxing
- ✓ **Prevents directory traversal outside project root**
- ✓ `cd ../` outside project root throws error
- ✓ Path validation enforces project boundaries

**Key Test Cases:**
```typescript
test("basic", async () => {
  const result = await bash.execute({
    command: "echo 'test'",
    description: "Echo test message",
  }, ctx)
  expect(result.metadata.exit).toBe(0)
  expect(result.metadata.output).toContain("test")
})

test("cd ../ should fail outside of project root", async () => {
  expect(
    bash.execute({
      command: "cd ../",
      description: "Try to cd to parent directory",
    }, ctx)
  ).rejects.toThrow("This command references paths outside of")
})
```

**Why It Matters:**
Security is paramount when AI agents execute bash commands. This sandboxing prevents:
- Accessing files outside the project
- Accidentally running destructive commands system-wide
- Data exfiltration to unexpected locations
- Privilege escalation attacks

**Related Files:**
- `packages/opencode/src/tool/bash.ts` - Bash tool implementation

---

## Test Infrastructure

### Test Fixtures (`fixture/fixture.ts`)

**Purpose:** Provides utilities for creating temporary test environments.

**Key Function: `tmpdir()`**

```typescript
async function tmpdir<T>(options?: {
  git?: boolean
  init?: (dir: string) => Promise<T>
  dispose?: (dir: string) => Promise<T>
}) => {
  path: string          // Temporary directory path
  extra: T              // Custom return value from init
  [Symbol.asyncDispose] // Auto-cleanup with `await using`
}
```

**Features:**
- ✓ Creates isolated temporary directories
- ✓ Optional git repository initialization
- ✓ Custom initialization logic
- ✓ Automatic cleanup via `using` syntax (TC39 proposal)
- ✓ Thread-safe with random directory names

**Usage Pattern:**
```typescript
test("example", async () => {
  await using tmp = await tmpdir({ git: true })
  // tmp.path is ready to use
  // Automatically deleted after test
})
```

### Test Preload (`preload.ts`)

**Purpose:** Sets up global test environment.

**Configuration:**
```typescript
Log.init({
  print: false,   // Suppress log output during tests
  dev: true,      // Enable development mode
  level: "DEBUG", // Full logging for debugging
})
```

Ensures consistent logging behavior across all tests.

---

## Test Coverage Summary

### What's Well Tested ✅

1. **Configuration System** - Comprehensive coverage
   - File loading (JSON/JSONC)
   - Merging and precedence
   - Variable substitution
   - Schema validation
   - Migration logic

2. **Snapshot System** - Exhaustive edge case testing
   - 30+ test scenarios
   - Edge cases: unicode, symlinks, permissions
   - Concurrent operations
   - Error handling

3. **Security** - Critical sandboxing tested
   - Directory traversal prevention
   - Project boundary enforcement

4. **Parsing** - Robust file reference extraction
   - Multiple path formats
   - Edge case handling

### What Could Use More Testing ⚠️

1. **Bash Tool** - Only 2 tests
   - More command patterns needed
   - Environment variable handling
   - stdin/stdout/stderr streams
   - Background processes
   - Timeouts and signals
   - Command chaining (&&, ||, ;)

2. **Integration Tests** - Missing
   - End-to-end workflows
   - Multi-agent scenarios
   - Real LLM provider integration
   - Full session lifecycle

3. **Performance Tests** - Not present
   - Large file handling
   - Memory usage under load
   - Concurrent session handling

4. **Error Recovery** - Limited coverage
   - Network failures
   - Malformed API responses
   - Partial file writes
   - Disk space exhaustion

---

## Test Execution

### Running Tests

```bash
# Run all tests
cd packages/opencode
bun test

# Run specific test file
bun test test/config/config.test.ts

# Run with coverage (if configured)
bun test --coverage

# Run tests matching pattern
bun test --test-name-pattern "snapshot"
```

### Current Blockers

❌ **Cannot run tests** due to dependency installation failure (see ENVIRONMENT_TEST_REPORT.md)

Once dependencies are installed:
```bash
bun install          # Install dependencies
bun test             # Run test suite
bun turbo typecheck  # Type checking
```

---

## Test Quality Metrics

### Code Quality Indicators

**✅ Good Practices Observed:**
- Use of `describe` blocks for organization
- Clear, descriptive test names
- Proper setup/teardown with `using` syntax
- Edge case coverage
- Error condition testing
- Isolation between tests
- No shared mutable state

**Patterns Used:**
- Arrange-Act-Assert pattern
- Fixture-based testing
- Temporary environment isolation
- Git-based state management
- Type-safe test utilities

---

## Recommendations

### For Test Suite Improvement

1. **Increase Bash Tool Coverage**
   - Add tests for more command patterns
   - Test environment variable handling
   - Verify stdin/stdout behavior
   - Test signal handling and timeouts

2. **Add Integration Tests**
   - Full conversation flow tests
   - Multi-tool usage scenarios
   - Real provider integration (with mocks)
   - Error recovery paths

3. **Performance Benchmarks**
   - Snapshot creation time
   - Large repository handling
   - Memory usage profiling
   - Concurrent operation limits

4. **Property-Based Testing**
   - Use fast-check or similar for:
     - Configuration parsing
     - File path handling
     - Markdown parsing

### For Development Workflow

1. **Pre-commit Testing**
   - Already have Husky configured
   - Could add: `bun test` to pre-commit hook
   - Consider: Only test affected packages

2. **CI/CD Integration**
   - Run full test suite on PR
   - Report coverage metrics
   - Block merge on test failures

3. **Test Documentation**
   - Add examples of running tests to README
   - Document test writing guidelines
   - Create test template for new features

---

## Key Insights from Test Analysis

### 1. Security-First Design
The extensive snapshot and bash sandboxing tests show OpenCode prioritizes user safety:
- AI can't escape project boundaries
- All changes are trackable and reversible
- Explicit validation of security boundaries

### 2. Configuration Flexibility
The config tests reveal a powerful, flexible system:
- Multiple config sources with clear precedence
- Environment-specific customization via variables
- Backward compatibility through migration
- Extensibility via agents and commands

### 3. Production-Ready Quality
The test suite demonstrates production quality:
- Edge cases thoroughly covered
- Unicode and internationalization handled
- Concurrent operations considered
- Error states explicitly tested

### 4. Bun-First Architecture
Tests leverage Bun-specific features:
- Native test runner (fast execution)
- `using` syntax for resource management
- Built-in shell execution via `$`
- TypeScript-first development

---

## Conclusion

The OpenCode test suite is **well-designed and comprehensive** for core functionality. The snapshot system in particular shows exceptional attention to edge cases. Security testing demonstrates a security-conscious approach.

**Strengths:**
- ✅ Core functionality well tested
- ✅ Security boundaries validated
- ✅ Edge cases thoroughly covered
- ✅ Modern testing practices

**Opportunities:**
- ⚠️ Expand bash tool test coverage
- ⚠️ Add integration tests
- ⚠️ Include performance benchmarks
- ⚠️ Consider property-based testing

**Overall Grade: A-**

The test suite provides confidence in core functionality, with room for expansion in integration testing and tool coverage.

---

**Report Generated:** 2025-12-12
**Analyzed By:** Claude Code Environment Testing
**Test Files Reviewed:** 5
**Test Cases Counted:** 50+
**Lines of Test Code:** ~1,500+
