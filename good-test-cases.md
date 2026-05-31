# Good Test Case Design

**Load this reference when:** designing new tests, reviewing existing tests, or questioning the quality of a test case.

> **Which guide?** This file covers function-level tests — one C function,
> scalar parameters, return value assertions. If you're testing a whole module
> with structured input (packets, config text, event sequences), use
> `references/component-test-design.md` instead.

## Core Principle

**Test behavior, not implementation.** If you change the implementation without changing behavior, good tests stay green. If you change behavior without changing implementation, good tests catch it.

## Quick Self-Check

| # | Question | Expected Answer |
|---|----------|-----------------|
| 1 | Does this test call the real function or a mock stub? | Real function |
| 2 | If you rename the function under test, does the assertion's meaning change? | No (assertions verify results, not internals) |
| 3 | Can this test run independently, without depending on side effects from other tests? | Yes |
| 4 | Run it in CI 1000 times — same result every time? | Yes |
| 5 | Do you read the test name and immediately know the requirement? | Yes |

If any answer is "no," the test case needs work.

## Six Dimensions of Good Tests

| Dimension | Good | Bad | Why It Matters |
|-----------|------|-----|----------------|
| **Focused** | One test verifies one behavior | `TEST(ParserTest, ValidatesFormatAndEncoding)` — "and" means split it | When it fails, you know exactly what broke |
| **Independent** | Tests don't share mutable state or depend on run order | Global variable bleed, passes solo but fails in suite | Can't trust results, can't parallelize |
| **Reproducible** | Same result every run | Depends on current time, random numbers, network state | Is the failure a real bug or a clock tick? |
| **Readable** | Test name is the specification | `TEST(FooTest, Test1)` | Will you understand it in six months? Will your teammate? |
| **Resilient** | Refactoring the implementation doesn't break the test | Renaming an internal variable breaks the test | Fear of refactoring = accumulating tech debt |
| **Fast** | Under 100ms | Over 1 second | Slow tests don't get run = worthless |

## Boundary Conditions Checklist for C

Always cover at minimum:

| Type | Boundary Values |
|------|-----------------|
| Pointer | `NULL` |
| Integer | `0`, `-1`, `INT_MIN`, `INT_MAX` |
| String | `""` (empty), `"A"` (single char), oversized string |
| Array/Buffer | Empty, single element, exactly full, overflow by one |
| Struct | `{0}`, partial initialization, circular reference |
| `size_t` | `0`, `1`, `SIZE_MAX` |
| Enum | First value, last value, out-of-range integer |
| Union | Test each member as the active field |
| Function pointer | `NULL`, self as callback |

## Test Structure: Given-When-Then

Originates from BDD (Dan North). Every clean test should show three distinct sections:

```c
TEST(MessageParserTest, ReportsErrorOnTruncatedLengthField) {
    /* Given — initial state */
    const unsigned char wire[] = {0x05, 0x00, 0x48}; /* claims 5 bytes, only 3 present */
    Message msg = {0};

    /* When — trigger action */
    ParseResult result = message_parse(wire, sizeof(wire), &msg);

    /* Then — expected outcome */
    EXPECT_EQ(result, PARSE_ERR_TRUNCATED);
    EXPECT_EQ(msg.type, 0);  /* parsing failed, fields remain zero */
}
```

Separate the three sections with blank lines. If you can't write a clear Given-When-Then structure, either the test is too vague or it's testing multiple behaviors.

## Triangulation in C Tests

Verify the same behavior two different ways (from *Pragmatic Unit Testing*, Hunt & Thomas):

```c
/* Forward: computed result should be correct */
result = calc_checksum(data, len);
EXPECT_EQ(result, expected);

/* Reverse: append checksum, recompute should yield 0 */
data_with_sum = append_checksum(data, len, result);
EXPECT_EQ(calc_checksum(data_with_sum, len + 2), 0);
```

Triangulation is much stronger than a single assertion — you might guess the expected value by coincidence, but both forward and reverse passing simultaneously is nearly impossible by luck.

## Test Double Decision Tree

```
Can the test run with real code?
  ├── Yes → Use real code
  └── No (too slow / external dependency / uncontrollable)
        ├── Can you inject a double via function pointer?
        │     └── Yes → Function pointer injection (preferred)
        │              e.g., config_load(path, out, fake_loader)
        └── No (static function, third-party lib, legacy code with no seam)
              └── Use MOCKER (last resort)
```

Rule: **function pointer injection > link substitution > MOCKER**. Each step to the right doubles the test's fragility and cost.

Reference: James Grenning, *Test-Driven Development for Embedded C*, Chapters 5-6.

## Naming Convention

```
TEST(ModuleUnderTest, Scenario_ExpectedBehavior)
```

Good names are living documentation:

```c
TEST(BufferTest, WriteFailsWhenBufferFull)
TEST(BufferTest, ReadReturnsZeroWhenEmpty)
TEST(BufferTest, WrapAroundOverwritesOldestData)
```

Bad names:

```c
TEST(BufferTest, TestBuffer1)       // meaningless
TEST(BufferTest, BufferLogic)       // what logic?
TEST(BufferTest, WriteRead)         // two behaviors in one test
```

## Industry Guidance

| Framework/Principle | Source | Core Idea |
|---------------------|--------|-----------|
| **F.I.R.S.T.** | *Clean Code*, Bob Martin | Fast / Isolated / Repeatable / Self-validating / Timely |
| **Given-When-Then** | Dan North, *Growing Object-Oriented Software, Guided by Tests* | Forces behavior-driven thinking |
| **Right-BICEP** | *Pragmatic Unit Testing*, Hunt & Thomas | Right / Boundary / Inverse / Cross-check / Error / Performance |
| **CORRECT Boundaries** | Same as above | Conformance / Ordering / Range / Reference / Existence / Cardinality / Time |
| **TDD for Embedded C** | James Grenning | Function pointer injection, dual-target compilation (C prod + C++ test) |
| **Arrange-Act-Assert** | Bill Wake | Three-section structure, equivalent to Given-When-Then |

## References

- Beck, K. *Test-Driven Development: By Example*. Addison-Wesley, 2002.
- Freeman, S. & Pryce, N. *Growing Object-Oriented Software, Guided by Tests*. Addison-Wesley, 2009.
- Grenning, J. *Test-Driven Development for Embedded C*. Pragmatic Bookshelf, 2011.
- Hunt, A. & Thomas, D. *Pragmatic Unit Testing in Java with JUnit*. Pragmatic Bookshelf, 2003.
- Martin, R. C. *Clean Code*. Prentice Hall, 2008.
