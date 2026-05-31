# Testing Anti-Patterns

**Load this reference when:** writing or changing tests, adding mocks, or tempted to add test-only code to production C files.

## Overview

Tests must verify real behavior, not mock behavior. Mocks are a means to isolate, not the thing being tested.

**Core principle:** Test what the code does, not what the mocks do.

**Following strict TDD prevents these anti-patterns.**

## Contents

- [Anti-Pattern 1: Testing Mock Behavior](#anti-pattern-1-testing-mock-behavior)
- [Anti-Pattern 2: Test-Only Code in Production](#anti-pattern-2-test-only-code-in-production)
- [Anti-Pattern 3: Mocking Without Understanding](#anti-pattern-3-mocking-without-understanding)
- [Anti-Pattern 4: Incomplete Struct Initialization](#anti-pattern-4-incomplete-struct-initialization)
- [Anti-Pattern 5: Integration Tests as Afterthought](#anti-pattern-5-integration-tests-as-afterthought)
- [When Mocks Become Too Complex](#when-mocks-become-too-complex)
- [C-Specific Mocking Patterns](#c-specific-mocking-patterns)
- [TDD Prevents These Anti-Patterns](#tdd-prevents-these-anti-patterns)
- [Quick Reference](#quick-reference)
- [Red Flags](#red-flags)
- [The Bottom Line](#the-bottom-line)

## The Iron Laws

```
1. NEVER test mock behavior
2. NEVER add test-only code to production C files
3. NEVER mock without understanding dependencies
```

## Anti-Pattern 1: Testing Mock Behavior

**The violation:**
```cpp
// ❌ BAD: Testing that the mock was called
TEST(ConfigTest, LoadsConfig) {
    MOCKER(read_config_file)
        .stubs()
        .will(returnValue(0));

    int result = app_init();

    // Verifying mock was called — doesn't test real behavior
    EXPECT_EQ(global_read_config_file_call_count, 1);
    EXPECT_EQ(result, 0);
}
```

**Why this is wrong:**
- You're verifying the mock was called, not that `app_init()` actually initialized correctly
- Tells you nothing about real behavior — mock call count ≠ correct initialization
- Test passes when mock is invoked, fails when it's not — never tests actual logic

**your human partner's correction:** "Are we testing the behavior of a mock?"

**The fix:**
```cpp
// ✅ GOOD: Test real behavior or don't mock it
TEST(ConfigTest, ParsesServerAddress) {
    const char *raw = "server=127.0.0.1:8080";

    ServerConfig cfg = {0};
    int result = parse_server_config(raw, &cfg);

    EXPECT_EQ(result, 0);
    // Verify actual downstream behavior, not mock invocation
    EXPECT_STREQ(cfg.host, "127.0.0.1");
    EXPECT_EQ(cfg.port, 8080);
}

// OR if parse_server_config must be mocked for isolation:
// Don't assert on the mock — test caller's behavior with config present
```

### Gate Function

```
BEFORE asserting on any MOCKER call count or mock parameter:
  Ask: "Am I testing real logic or just mock invocation?"

  IF testing mock invocation:
    STOP - Delete the assertion or unmock the function

  Test real behavior instead
```

## Anti-Pattern 2: Test-Only Code in Production

**The violation:**
```cpp
// ❌ BAD: cache_invalidate_all() only used in tests
/* user_cache.c */
#ifdef TEST
void cache_invalidate_all(UserCache *c) {  // Looks like production API!
    for (int i = 0; i < c->num_entries; i++) {
        free(c->entries[i].name);
        free(c->entries[i].email);
    }
    c->num_entries = 0;
}
#endif

/* tests */
TEST(UserCacheTest, LookupCachesResult) {
    UserCache cache = {0};
    cache_load_users(&cache, "alice,bob,eve");
    // ... test ...
    cache_invalidate_all(&cache);  // Only exists under #ifdef TEST
}
```

**Why this is wrong:**
- Production code polluted with test-only code paths
- `#ifdef TEST` changes the compiled binary between test and production builds — you're not testing what ships
- Dangerous if accidentally called in production (when compiled with TEST defined)
- Violates YAGNI and separation of concerns

**The fix:**
```cpp
/* ✅ GOOD: Test utilities handle test cleanup */
/* test_utils/test_cache_utils.c */
void test_cache_reset(UserCache *c) {
    for (int i = 0; i < c->num_entries; i++) {
        free(c->entries[i].name);
        free(c->entries[i].email);
    }
    c->num_entries = 0;
}

/* tests */
TEST(UserCacheTest, LookupCachesResult) {
    UserCache cache = {0};
    cache_load_users(&cache, "alice,bob,eve");
    // ... test ...
    test_cache_reset(&cache);  // Lives in test code, not production
}
```

### Gate Function

```
BEFORE adding any #ifdef TEST block or function to production C files:
  Ask: "Is this only used by tests?"

  IF yes:
    STOP - Don't add it
    Put it in test utilities instead

  Ask: "Would this code ship in a release build?"

  IF no:
    STOP - You're testing a different binary than what ships
```

## Anti-Pattern 3: Mocking Without Understanding

**The violation:**
```cpp
// ❌ BAD: Mock breaks test logic
TEST(ServerTest, DetectsDuplicateServer) {
    // Mock prevents config write that test depends on!
    MOCKER(save_server_config)
        .stubs()
        .will(returnValue(0));

    add_server(&cfg, "server1");    // Config IS written (or should be)
    add_server(&cfg, "server1");    // Should return ERR_DUPLICATE — but won't!
    // Because save_server_config was mocked away, the duplicate
    // detection logic (which checks the saved config) never fires.
}
```

**Why this is wrong:**
- Mocked function had side effect test depended on (writing config to disk/memory)
- Over-mocking to "be safe" breaks actual behavior
- Test passes for wrong reason or fails mysteriously

**The fix:**
```cpp
/* ✅ GOOD: Mock at correct level */
TEST(ServerTest, DetectsDuplicateServer) {
    // Mock the slow part (network), preserve behavior test needs (config write)
    MOCKER(send_handshake)
        .stubs()
        .will(returnValue(0));

    add_server(&cfg, "server1");    // Config written ✓
    add_server(&cfg, "server1");    // ERR_DUPLICATE detected ✓
}

> **Design note:** The real fix is making `add_server` check an in-memory data
> structure (e.g., a hash table of server names) for duplicates before calling
> `save_server_config`. Mocking the wrong layer is a symptom of bad design —
> the test should never depend on I/O side effects for core business logic.
```

### Gate Function

```
BEFORE mocking any function:
  STOP - Don't mock yet

  1. Ask: "What side effects does the real function have?"
  2. Ask: "Does this test depend on any of those side effects?"
  3. Ask: "Do I fully understand what this test needs?"

  IF depends on side effects:
    Mock at lower level (the actual slow/external operation)
    OR use test doubles that preserve necessary behavior
    NOT the high-level function the test depends on

  IF unsure what test depends on:
    Run test with real implementation FIRST
    Observe what actually needs to happen
    THEN add minimal mocking at the right level

  Red flags:
    - "I'll mock this to be safe"
    - "This might be slow, better mock it"
    - Mocking without understanding the dependency chain
```

## Anti-Pattern 4: Incomplete Struct Initialization

**The violation:**
```c
// ❌ BAD: Partial struct — only fields you think you need
Response resp = {
    .status = OK,
    .data = {.user_id = 123, .name = "Alice"}
    // Missing: .metadata that downstream code uses
};

// Later: segfault or garbage when code reads resp.metadata.request_id
```

**Why this is wrong:**
- **Partial initialization hides structural assumptions** — You only initialized fields you know about
- **Downstream code may depend on fields you didn't set** — Silent failures or crashes
- **Tests pass but integration fails** — Mock struct incomplete, real struct complete
- **False confidence** — Test proves nothing about real behavior

**The Iron Rule:** Initialize the COMPLETE struct as it exists in reality, not just fields your immediate test uses. Use designated initializers for all fields, or `memset` + explicit field overrides.

**The fix:**
```c
// ✅ GOOD: Mirror real struct completeness
Response resp = {
    .status = OK,
    .data = {
        .user_id = 123,
        .name = "Alice"
    },
    .metadata = {
        .request_id = "req-789",
        .timestamp = 1234567890
    }
    // All fields real code expects
};

// OR when struct is too large:
Response resp;
memset(&resp, 0, sizeof(resp));
resp.status = OK;
resp.data.user_id = 123;
strncpy(resp.data.name, "Alice", sizeof(resp.data.name));
resp.metadata.request_id = "req-789";
resp.metadata.timestamp = 1234567890;
```

### Gate Function

```
BEFORE creating mock structs:
  Check: "What fields does the real struct contain?"

  Actions:
    1. Examine the struct definition in the header file
    2. Include ALL fields downstream code might consume
    3. Verify mock struct matches real struct schema completely

  Critical:
    If you're initializing a struct for testing, you must understand
    the ENTIRE struct layout. Partial initialization fails silently
    when code depends on omitted fields.

  If uncertain: Initialize all fields to known values (zero/NULL/empty)
```

## Anti-Pattern 5: Integration Tests as Afterthought

**The violation:**
```
✅ Implementation complete
❌ No tests written
"Ready for testing"
```

**Why this is wrong:**
- Testing is part of implementation, not optional follow-up
- TDD would have caught this
- Can't claim complete without tests

**The fix:**
```
TDD cycle:
1. Write failing test
2. Implement to pass
3. Refactor
4. THEN claim complete
```

## When Mocks Become Too Complex

**Warning signs:**
- Mock setup longer than test logic
- Mocking everything to make test pass
- Mocker stubs missing return paths real functions have
- Test breaks when mock changes

**your human partner's question:** "Do we need to be using a mock here?"

**Consider:** Integration tests with real components (in-memory buffers, pre-populated structs, pipe-based IPC) often simpler than complex mocks.

## C-Specific Mocking Patterns

> **Note**: This is the only section in this file where mockcpp usage is shown
> as a solution. All other MOCKER() appearances in prior sections are RED FLAGS —
> patterns to avoid. The hierarchy is: **function pointer injection → test
> utilities → MOCKER (last resort).**

### Function Pointer Injection (prefer over mockcpp when possible)

```cpp
/* ✅ GOOD: Design for testability with function pointers */

/* config.h */
typedef int (*config_loader_t)(const char *path, Config *out);

/* config.c */
int config_load(const char *path, Config *out, config_loader_t loader) {
    return loader(path, out);  // Caller provides loader
}

/* tests */
static int fake_loader(const char *path, Config *out) {
    out->port = 8080;
    return 0;
}

TEST(ConfigTest, LoadsWithFakeLoader) {
    Config cfg = {0};
    int result = config_load("any", &cfg, fake_loader);
    EXPECT_EQ(result, 0);
    EXPECT_EQ(cfg.port, 8080);
}
```

### When to Use mockcpp

Reserve mockcpp for functions you CANNOT inject via function pointers:
- Static functions in other translation units
- Platform/project constraints preventing DI
- Legacy code without test seams

```cpp
// mockcpp used as last resort, not first choice:
MOCKER(platform_sleep)    // Can't inject because it's a platform call
    .stubs()
    .will(returnValue(0));
```

## TDD Prevents These Anti-Patterns

**Why TDD helps:**
1. **Write test first** → Forces you to think about what you're actually testing
2. **Watch it fail** → Confirms test tests real behavior, not mocks
3. **Minimal implementation** → No test-only code creeps in
4. **Real dependencies** → You see what the test actually needs before mocking

**If you're testing mock behavior, you violated TDD** — you added mocks without watching test fail against real code first.

## Quick Reference

| Anti-Pattern | Fix |
|--------------|-----|
| Assert on MOCKER call counts | Test real behavior or unmock it |
| `#ifdef TEST` blocks in production | Move to test utility files |
| Mock without understanding | Understand dependencies first, mock minimally |
| Incomplete struct initialization | Mirror real struct completely |
| Tests as afterthought | TDD — tests first |
| Over-complex mocks | Prefer function pointer injection or integration tests |

## Red Flags

- Assertion checks for mock call counts or mock parameters
- `#ifdef TEST` / `#ifndef UNIT_TEST` / `#ifdef DEBUG` in `.c` files
- Mock setup is >50% of test body
- Test fails when you remove MOCKER()
- Can't explain why mock is needed
- Mocking "just to be safe"

## The Bottom Line

**Mocks are tools to isolate, not things to test.**

If TDD reveals you're testing mock behavior, you've gone wrong.

Fix: Test real behavior or question why you're mocking at all.
