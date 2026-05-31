# Systematically Adding Tests to a Legacy C Codebase

**Load this reference when:** facing a large untested C codebase and needing a plan
to add tests systematically — not ad-hoc, not all at once, but with measurable progress.

## Core Principle

Testing legacy code is an investment with diminishing returns. The goal is not
100% coverage. The goal is **enough safety to change code without fear**.

> "Code without tests is bad code. It doesn't matter how well written it is;
>  it doesn't matter how pretty or object-oriented or well-encapsulated it is.
>  With tests, we can change the behavior of our code quickly and verifiably.
>  Without them, we really don't know if our code is getting better or worse."
>  — Michael Feathers, *Working Effectively with Legacy Code*

---

## Phase 0: Triage — Know What You're Dealing With

Before writing a single test, gather data:

| Action | Command / Tool | Output |
|--------|---------------|--------|
| Count lines and functions | `cloc .` or `tokei .` | Scale: how big is this? |
| Existing coverage (if any) | `cmake -DCMAKE_BUILD_TYPE=Debug .. && make && gcov -b *.gcno` | Baseline |
| Change hotspots | `git log --format=format:%H --name-only \| sort \| uniq -c \| sort -nr \| head -50` | Files that change most often |
| Dead code | Compile with `-Wunused-function -Wunused-variable` | Functions that may not even need tests |

**Decision rule:** Start with the hottest file that has zero tests. Not the
largest file. Not the most complex file. The file that changes most often.

---

## Phase 1: Build Skeleton — Make One Empty Test Compile

The biggest technical hurdle with legacy C is not the tests — it's the build.

```
project/
├── src/           ← production .c files (don't touch)
├── include/       ← production .h files (don't touch)
└── tests/
    ├── CMakeLists.txt
    ├── test_utils/
    │   └── test_helpers.c       ← builder functions, fake data
    └── module_a/
        └── test_foo.cpp
```

**CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.14)
project(myproject_tests C CXX)

find_package(GTest REQUIRED)
include(GoogleTest)

add_library(prod_lib STATIC
    ../src/module_a.c
    ../src/module_b.c
    # ... all production .c files
)
target_include_directories(prod_lib PUBLIC ../include)

add_executable(module_a_test
    module_a/test_foo.cpp
)
target_include_directories(module_a_test PRIVATE ../include)
target_link_libraries(module_a_test GTest::gtest prod_lib)
gtest_discover_tests(module_a_test)
```

**Goal for this phase:** `cmake --build build && ./build/module_a_test` prints
`[  PASSED  ] 0 tests`. Do not move to Phase 2 until this works.

**Common blockers:**
- Circular `#include` in legacy headers → fix with forward declarations
- Global mutable state in headers → wrap in `extern` + define once in `.c`
- Production code calls `exit()` / `abort()` on error → for tests, redirect via
  a function pointer or write a characterization test that expects the crash
  (GTest `EXPECT_DEATH`)

---

## Phase 2: Prioritize — Risk-Weighted Test Order

Do NOT test files in alphabetical order. Prioritize by risk:

```
Priority = ChangeFrequency × BusinessCriticality ÷ ExistingCoverage
```

### Tier 1 — High Priority (start here)

| Criteria | Examples |
|----------|----------|
| Changed >5 times in last 3 months (`git log`) | Config parser, protocol handler |
| Parses user input / network data / file formats | CSV parser, HTTP header parser, message deserializer |
| Core business logic | Billing calculation, access control, state machine transitions |
| Known bug magnets | Functions mentioned in commit messages with "fix" / "bug" / "crash" |

### Tier 2 — Medium Priority

| Criteria | Examples |
|----------|----------|
| Error handling paths (not yet tested in Tier 1) | `if (pkt == NULL) return -1;` branches |
| Utility functions with many edge cases | String formatters, checksum calculators, sort/merge helpers |
| Functions called from Tier 1 code (indirect coverage is partial) | Internal helpers that Tier 1 tests happen to exercise |

### Tier 3 — Low Priority (possibly never)

| Criteria | Examples |
|----------|----------|
| Simple getter/setter | `int get_port() { return port; }` |
| Glue / boilerplate | Init functions that zero a struct, trivial wrappers |
| Stable for >2 years, never changed | If it's not broken and nobody touches it, testing has low ROI |

---

## Phase 3: Break Dependencies — Make Code Testable

Legacy C code's #1 obstacle is not "how to test" — it's "code is untestable."
Common patterns and their fixes:

| Problem | Pattern You'll See | Fix |
|---------|-------------------|-----|
| Hardcoded I/O | `fopen(path, "r")` mid-function | Accept `FILE*` parameter; caller provides `fmemopen` in tests |
| Hardcoded time | `time(NULL)` mid-function | Inject `uint64_t (*clock)(void *ctx)` |
| Hardcoded allocator | `malloc` / `free` everywhere | For now: test with valgrind. Later: inject allocator function pointers |
| Global mutable state | `static int g_state = 0;` at file scope | Write characterization test to lock current behavior; then eliminate |
| Deep call chains | `a() → b() → c() → d()` (10 layers) | Test leaf functions first (bottom-up). They have fewest dependencies |
| 200-line `static` function | Can't test directly | Extract to its own `.c/.h` module with public interface |
| `exit()` / `abort()` on error | Untestable crash | Replace with error-return pattern; use `EXPECT_DEATH` in the meantime |

**Golden rule:** Separate "breaking dependencies" from "adding tests" into
separate commits. Commit A: pure refactor, no behavior change, all existing tests
green. Commit B: add new tests. This keeps `git bisect` usable.

### Breaking Dependencies Without Breaking Behavior

When you extract a `static` helper into a public function, do it mechanically:

```c
/* BEFORE (static, untestable) */
static int validate_header(const uint8_t *data, size_t len) {
    /* 30 lines of logic */
}

/* AFTER — Step 1 (refactor only, commit A) */
/* header_validator.h */
#ifndef HEADER_VALIDATOR_H
#define HEADER_VALIDATOR_H
#include <stdint.h>
#include <stddef.h>
int header_validate(const uint8_t *data, size_t len);
#endif

/* header_validator.c — exact same code, now public */
#include "header_validator.h"
int header_validate(const uint8_t *data, size_t len) {
    /* 30 lines of logic — identical, not rewritten */
}

/* original.c — delegate to new module */
#include "header_validator.h"
/* delete static validate_header, replace call site with header_validate */
```

Now `git diff` shows a pure extraction. Existing callers still work. Next commit
adds `test_header_validator.cpp`. If the test reveals a bug, that's a third commit.

---

## Phase 4: Per-Function Test Rhythm

Once a function is isolated and testable, follow this rhythm:

```
1. READ the function source. Understand inputs, outputs, side effects.
2. Write a CHARACTERIZATION test → run → GREEN (lock current behavior)
3. Add BOUNDARY tests (NULL, 0, empty, overflow, error paths) → run → GREEN
4. Add TRIANGULATION if applicable (forward + reverse verification)
5. Run under valgrind / AddressSanitizer → confirm no leaks or UB
6. Mark function DONE → move to next
```

### Characterization Test Template

```cpp
/*
 * Characterization test: documents what parse_config currently returns,
 * not what it "should" return. If this test ever changes during refactoring,
 * you accidentally changed behavior.
 */
TEST(ConfigParserCharTest, HandlesKeyValuePair_CurrentBehavior) {
    const char *input = "port=8080";
    Config cfg = {0};

    int rc = parse_config(input, &cfg);

    // These assertions describe ACTUAL behavior, bugs and all.
    EXPECT_EQ(rc, 0);
    EXPECT_EQ(cfg.port, 8080);
    // If parse_config ignores trailing spaces, DON'T add that expectation.
    // If parse_config returns -1 for empty input instead of 0, DO assert that.
}
```

### Boundary Test Checklist

Use this as a literal checklist for each function. From `good-test-cases.md`:

> | Type | Values |
> |------|--------|
> | Pointer | `NULL` |
> | Integer | `0`, `-1`, `INT_MIN`, `INT_MAX` |
> | String | `""`, `"A"`, oversized (>buffer) |
> | Buffer/Array | Empty, single element, exactly full, overflow by one |
> | Struct | `{0}`, partial init, circular reference |
> | `size_t` | `0`, `1`, `SIZE_MAX` |
> | Enum | First value, last value, out-of-range cast |
> | Union | Each member as the active field |
> | Function pointer | `NULL`, self as callback |

---

## Phase 5: Measure Progress — Stay Motivated

Without measurement, legacy testing efforts stall. Pick at least two metrics:

| Metric | How to Track | Target |
|--------|-------------|--------|
| Functions covered / total functions | `gcov -b` → parse `*.gcov` output | >70% for Tier 1+2 |
| Line coverage trend | `lcov --rc lcov_branch_coverage=1` weekly, generate HTML | Trending up, not flat |
| High-risk functions remaining | Manual `TODO.md` checklist | Drive to zero for Tier 1 |
| Bug escape rate | Count of bugs found in production in tested vs untested modules | Should drop in tested modules |

**Do not target 100% coverage.** Industry data consistently shows diminishing
returns beyond 70-80% line coverage. The last 20% costs more than the first 80%.

---

## When to Stop

You've done enough when:

- [ ] Tier 1 functions all have characterization + boundary tests
- [ ] New bugs can be reproduced as a failing test within 5 minutes
- [ ] Refactoring feels safe: you change code and trust the test suite will catch regressions
- [ ] The team adds tests for every new feature and bug fix (TDD going forward)

---

## Reference

- Feathers, M. *Working Effectively with Legacy Code*. Prentice Hall, 2004.
  Chapters 6 ("I Don't Have Much Time and I Have to Change It"), 12 ("I Need to
  Make Many Changes in One Area"), and 13 ("I Need to Make a Change but I Don't
  Know What Tests to Write") are directly applicable to this document.
- Grenning, J. *Test-Driven Development for Embedded C*. Pragmatic Bookshelf, 2011.
  Chapter 7 ("Introducing TDD into an Existing Codebase") covers legacy C specifically.
- `testing-anti-patterns.md` — Anti-Pattern 5 (Integration Tests as Afterthought)
  and the decision table for `static` function extraction.
- `testable-design.md` — Module boundary design and dependency inversion patterns
  used in Phase 3.
