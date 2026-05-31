# Designing Testable C Code

**Load this reference when:** designing a new module interface, refactoring
legacy code for testability, or unsure how to structure code so tests can
verify it without mockcpp.

## Contents

- [Module Boundary Design](#module-boundary-design)
- [Dependency Inversion](#dependency-inversion)
- [Parameter Design](#parameter-design)
- [Testability vs Static Cohesion](#testability-vs-static-cohesion)
- [Time, I/O, and Non-Determinism](#time-io-and-non-determinism)

## Module Boundary Design

Testable C code starts with clean module boundaries. A module is a `.c` file
plus its public `.h` header. The `.h` is the contract; the `.c` is the
implementation. Tests only see the `.h`.

**Good:**
```c
/* ringbuf.h — public interface */
typedef struct RingBuf RingBuf;       /* opaque — tests can't peek inside */

RingBuf* ringbuf_create(size_t capacity);
int      ringbuf_write(RingBuf *rb, char data);
int      ringbuf_read(RingBuf *rb, char *out);
void     ringbuf_destroy(RingBuf *rb);
```

**Bad:**
```c
/* ringbuf.h — leaks implementation */
struct RingBuf {                      /* tests will couple to internals */
    char *buf;
    size_t head, tail, count;
    size_t capacity;
};
/* ... function declarations ... */
```

Opaque structs force tests to verify behavior through the public API, exactly
as production code uses it. If you need internal state to set up a test
scenario, add a constructor or builder function — don't expose the struct.

**Builder pattern for test setup:**
```c
/* test helper — lives in test code, not production */
RingBuf* test_ringbuf_with_data(const char *data, size_t len) {
    RingBuf *rb = ringbuf_create(len * 2);
    for (size_t i = 0; i < len; i++) ringbuf_write(rb, data[i]);
    return rb;
}
```

## Dependency Inversion

When a function depends on something slow, external, or non-deterministic,
inject the dependency via a function pointer rather than hardcoding the call.

**Bad — hardcoded dependency:**
```c
int process_data(const char *path) {
    char *raw = read_file_from_disk(path);   /* Untestable: requires disk */
    int result = parse(raw);
    free(raw);
    return result;
}
```

**Good — injected dependency:**
```c
typedef char* (*reader_t)(const char *path, void *ctx);

int process_data(const char *path, reader_t reader, void *ctx) {
    char *raw = reader(path, ctx);            /* Test injects in-memory reader */
    int result = parse(raw);
    free(raw);
    return result;
}
```

**Opaque struct + vtable** (when multiple functions share dependencies):
```c
/* protocol.h */
typedef struct Protocol Protocol;     /* opaque */

typedef struct {
    int  (*send)(const uint8_t *data, size_t len, void *ctx);
    int  (*recv)(uint8_t *buf, size_t len, void *ctx);
    void *ctx;
} Transport;

Protocol* protocol_create(const Transport *transport);
int        protocol_handshake(Protocol *p);
void       protocol_destroy(Protocol *p);
```

This lets tests inject a fake `Transport` with in-memory send/recv. Production
code injects the real socket implementation. The `Protocol` module doesn't know
or care which it gets.

## Parameter Design

How you pass data into a function directly affects testability.

| Pattern | When | Testability |
|---------|------|-------------|
| `int fn(Data *out)` | Caller owns output memory | ★★★  Tests allocate local struct |
| `Data* fn(void)` | Callee allocates, caller frees | ★★☆  Tests must remember to free |
| `int fn(Data in)` | Value semantics, small struct | ★★★  Self-contained, no side effects |
| `int fn(Data *in, Data *out)` | Transform pipeline | ★★★  Clear input/output boundary |
| `int fn(void *ctx, Data *out)` | Generic callback context | ★★☆  Tests cast ctx correctly |

Prefer `*out` parameters over returned pointers. Prefer small by-value structs
over `*in` for immutable inputs. Prefer explicit error codes (`0` = success,
negative = error) over side-channel error reporting (global `errno`, logging).

## Testability vs Static Cohesion

`static` functions in `.c` files are invisible to tests. Sometimes you want to
test them directly; sometimes you should test them indirectly.

**Decision table:**

| Situation | Action |
|-----------|--------|
| `static` helper is simple (1-2 lines) | Test indirectly through public API |
| `static` helper is complex (10+ lines) | Extract into a separate module with public `.h` |
| `static` helper is used by ONE public function | Test through that function |
| `static` helper is used by MANY public functions | Extract it — the cohesion signal is real |
| `static` helper has tricky edge cases | Expose via a function pointer or #ifdef TEST_TESTABLE macro |

**When to extract:**
```c
/* BEFORE: complex static, untestable directly */
static int validate_header(const uint8_t *data, size_t len) {
    /* 30 lines of CRC check, version negotiation, flag parsing... */
}
/* test it? can't — it's static */

/* AFTER: extracted to its own module */
/* header_validator.h */
int header_validate(const uint8_t *data, size_t len);

/* header_validator.c */
int header_validate(const uint8_t *data, size_t len) { /* ... */ }
/* test it directly ✓ */
```

Don't extract TRIVIAL helpers — one extra `.h`/`.c` pair per trivial function
creates more noise than value. Extract when the helper has its own edge cases,
error paths, or algorithmic complexity.

## Time, I/O, and Non-Determinism

Tests must be reproducible. Real time, real files, and real randomness break
reproducibility.

**Time:**
```c
/* Inject a clock function instead of calling time() directly */
typedef uint64_t (*clock_fn)(void *ctx);

int session_is_expired(Session *s, clock_fn now, void *ctx) {
    return (now(ctx) - s->created_at) > s->ttl_seconds;
}

/* Test: */
static uint64_t fake_clock(void *ctx) { return *(uint64_t*)ctx; }
uint64_t t = 1000;
EXPECT_FALSE(session_is_expired(&s, fake_clock, &t));
t = 2000;
EXPECT_TRUE(session_is_expired(&s, fake_clock, &t));
```

**Randomness:**
```c
/* Inject a random source */
typedef uint32_t (*rand_fn)(void *ctx);

int shuffle(int *arr, size_t len, rand_fn rng, void *ctx);

/* Test with deterministic sequence */
static uint32_t seq[] = {3, 1, 4, 1, 5};
static size_t   seq_i = 0;
static uint32_t fake_rand(void *ctx) { return seq[seq_i++]; }

int arr[] = {10, 20, 30, 40, 50};
seq_i = 0;
shuffle(arr, 5, fake_rand, NULL);
/* Verify deterministic output */
```

**File I/O:** Where possible, operate on `const char *` buffers or `FILE*`
handles that the caller provides, so tests can use `fmemopen` or
`tmpfile` instead of real paths.

## Reference

- Grenning, J. *Test-Driven Development for Embedded C*, Chapters 5-6.
- Feathers, M. *Working Effectively with Legacy Code*, Chapter 6 (I Don't
  Have Much Time and I Have to Change It).
