# Component-Level Test Design for C

**Load this reference when:** testing a module, protocol handler, or subsystem
that processes structured input — packets, messages, configuration blocks, event
sequences. If you're writing a test for a single pure function with simple
parameters, use the boundary checklist in `good-test-cases.md` instead.

## What Is a Component-Level Test?

| | Function-Level (Unit) | Component-Level (Integration) |
|---|---|---|
| **Test unit** | One C function | One module / subsystem (.c + .h) through its public API |
| **Input** | Scalar parameters (`int`, `char*`, `struct*`) | Structured data (binary packets, config text, message buffers, event sequences) |
| **Assertion target** | Return value, output parameter | Observable business behavior: parsed result, state transition, side effect on a peer component |
| **Mock boundary** | Functions called by the function under test | External I/O, hardware, or cross-component calls at the module perimeter |

The TDD cycle (RED-GREEN-REFACTOR) is identical regardless of test granularity.
The difference is *what you feed in* and *what you assert*.

---

## Core Principle

**Test through the public API with real data.** Construct the input, trigger the
component's processing, assert the observable outcome. Do not assert on internal
struct fields that are not part of the module's documented contract.

If a behavior change doesn't alter the observable output, the test should still
pass. If it does, the test should catch it.

---

## Scenario 1: Binary Protocol Packet Handler

**Component:** `pkt_handler.c` — receives raw bytes, validates header, dispatches
payload to upper layer.

**Test input:** A constructed byte buffer simulating what a socket would deliver.

**Test case:** Corrupted packet with length field claiming more data than the
buffer contains.

```cpp
TEST(PktHandlerTest, RejectsTruncatedPacket) {
    // Given — construct a fake wire packet
    uint8_t raw[] = {
        0xAB, 0xCD,              // magic
        0x00, 0x00, 0x00, 0x64,  // length field: claims 100 bytes payload
        0x01, 0x02, 0x03         // actual payload: only 3 bytes
    };
    struct pkt_handler *h = pkt_handler_create(NULL);

    // When — feed it to the handler
    int rc = pkt_handler_feed(h, raw, sizeof(raw));

    // Then — handler should reject, not crash or return partial data
    EXPECT_EQ(rc, PKT_ERR_TRUNCATED);
    pkt_handler_destroy(h);
}
```

**What's being tested:** The component's validation logic as a whole — header
parsing, length verification, error signaling. Not individual `parse_header()`
or `verify_length()` functions.

**Boundary inputs for this scenario:**

| Input variation | Test intent |
|-----------------|-------------|
| Empty buffer (`len=0`) | Does the handler reject gracefully? |
| Magic mismatch | Is unknown protocol detected? |
| Length = 0 | Zero-length payload — valid edge or protocol violation? |
| Length = exactly buffer remaining bytes | Exactly full — should succeed |
| Length > SIZE_MAX in buffer | Overflow defense |
| Checksum mismatch | Integrity check fires? |
| Two complete packets in one buffer | Back-to-back dispatch works? |

---

## Scenario 2: Configuration File Loader

**Component:** `config_loader.c` — reads a key=value config text, populates a
`Config` struct, reports errors by line.

**Test input:** A C string holding the config file content. No actual file on
disk — the loader accepts a buffer or `FILE*` (ideally via `fmemopen`).

**Test case:** Missing mandatory key.

```cpp
TEST(ConfigLoaderTest, ReportsMissingMandatoryKey) {
    // Given — config content missing "host"
    const char *content =
        "port=8080\n"
        "# host is not set\n"
        "timeout=30\n";
    Config cfg = {0};

    // When
    int rc = config_load_from_string(content, &cfg);

    // Then
    EXPECT_EQ(rc, CFG_ERR_MISSING_KEY);
    EXPECT_STREQ(cfg.err_key, "host");
    // port should NOT have been partially applied
    EXPECT_EQ(cfg.port, 0);
}
```

**Boundary inputs for this scenario:**

| Input variation | Test intent |
|-----------------|-------------|
| Empty string `""` | Empty config — default values applied? |
| Key with no `=` | Syntax error detection |
| Value missing (`key=\n`) | Empty string vs. error |
| Duplicate key (`host=A\nhost=B`) | Last-wins or error? |
| Value exceeding buffer size (10KB for a 64-byte field) | Truncation or rejection? |
| Unknown key | Silently ignored or error? |
| Whitespace around `=` (`key = value`) | Tolerant parsing? |
| Trailing newline absent (last line no `\n`) | Edge of file handling |
| BOM / UTF-8 illegal sequence | Encoding robustness |
| Include directive (`include other.conf`) | Recursion guard? Cycle detection? |

---

## Scenario 3: State Machine / Event-Driven Component

**Component:** `conn_fsm.c` — a connection state machine with states `IDLE →
CONNECTING → ESTABLISHED → CLOSED`. Receives events from a network layer.

**Test input:** A sequence of `Event` structs delivered via `conn_fsm_event()`.

**Test case:** Duplicate CONNECT event while already in CONNECTING state.

```cpp
TEST(ConnFsmTest, IgnoresDuplicateConnect) {
    // Given — a fresh FSM
    ConnFsm *fsm = conn_fsm_create();
    Event ev_connect = {.type = EV_CONNECT, .peer = 0};

    // When — send CONNECT twice
    conn_fsm_event(fsm, &ev_connect);
    conn_fsm_event(fsm, &ev_connect);

    // Then — still in CONNECTING, not crashed or double-counted
    EXPECT_EQ(conn_fsm_state(fsm), CONN_STATE_CONNECTING);
    EXPECT_EQ(conn_fsm_connect_attempts(fsm), 1);

    conn_fsm_destroy(fsm);
}
```

**Boundary inputs for this scenario:**

| Input variation | Test intent |
|-----------------|-------------|
| Events in wrong order (e.g., DATA before CONNECT) | Out-of-sequence defense |
| DISCONNECT while in IDLE | Double-close safety |
| DISCONNECT while in CONNECTING | Cancel mid-handshake |
| Timeout event after already ESTABLISHED | Stale event ignored? |
| Rapid sequential events (1000 events in tight loop) | Race / reentrancy |
| NULL event pointer | Crash protection |

---

## Scenario 4: Pipeline / Transform Component

**Component:** `msg_filter.c` — receives raw messages, passes them through a
chain of filters (dedup, rate-limit, ACL), outputs accepted messages to a sink.

**Test input:** A sequence of `Message` structs fed via `msg_filter_feed()`, with
a fake sink callback to capture output.

```cpp
static Message captured = {0};
static int capture_count = 0;

static void fake_sink(const Message *msg, void *ctx) {
    (void)ctx;
    captured = *msg;
    capture_count++;
}

TEST(MsgFilterTest, DedupDropsDuplicateWithinWindow) {
    MsgFilter *f = msg_filter_create(fake_sink, NULL);

    Message a = {.id = 42, .payload = "hello"};
    Message b = {.id = 42, .payload = "world"};  // same ID as a
    Message c = {.id = 43, .payload = "bye"};

    msg_filter_feed(f, &a);
    msg_filter_feed(f, &b);  // duplicate — should be dropped
    msg_filter_feed(f, &c);

    EXPECT_EQ(capture_count, 2);
    EXPECT_EQ(captured.id, 43);            // last accepted = c, not b
    EXPECT_STREQ(captured.payload, "bye");

    msg_filter_destroy(f);
}
```

**Boundary inputs for this scenario:**

| Input variation | Test intent |
|-----------------|-------------|
| First message to empty pipeline | Init path |
| All messages dropped by filter | Zero-output case |
| Single message with max-size payload | Buffer boundary |
| Interleaved filter stages: allow → deny → allow | State carryover between filters |
| Sink callback returns error | Feedback/backpressure handling |
| Feed with NULL message | Crash protection |

---

## Input Construction Patterns — Quick Reference

Use this table when designing test inputs for component-level tests:

| Input Type | Construct With | Boundary Examples |
|------------|---------------|-------------------|
| **Binary packet** | `uint8_t raw[] = {...}` | Truncated, magic mismatch, oversized length field, bad checksum, multiple packets in one buffer |
| **Text protocol message** | `const char *msg` | Empty line, missing delimiter, key with no value, oversized line, illegal encoding bytes |
| **Configuration text** | `const char *content` via `fmemopen` or buffer API | Empty file, missing mandatory key, out-of-range value, duplicate section, include loop |
| **Callback / event sequence** | Array of `Event` structs, loop calling `xxx_feed()` | Out-of-order arrival, duplicate/missing events, timeout after done, cancel mid-operation |
| **Resource lifecycle** | Repeated `xxx_create()` / `xxx_destroy()` calls | Double-init, use-after-destroy, create-destroy-create, NULL handle to every function |
| **Stream / chunked input** | Split one logical message across multiple `feed()` calls | 1-byte chunks, boundary exactly at edge of internal buffer, empty chunk between real data |

---

## Summary Checklist for a Component-Level Test

Before calling a component test "done":

- [ ] Input is constructed as structured data (byte buffer, config string, event structs), not individual scalar parameters
- [ ] Assertions verify observable business behavior, not internal struct fields
- [ ] Test covers at least: empty input, corrupted/malformed input, normal input, and one edge case from the boundary table above
- [ ] Test passes without the mock framework if the component has no real I/O — real code, not mock behavior
- [ ] Mock boundary is at the component perimeter (hardware, network, filesystem), not inside the component
