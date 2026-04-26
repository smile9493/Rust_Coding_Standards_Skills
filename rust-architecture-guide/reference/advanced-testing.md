# Advanced Testing & Quality Assurance

## Core Philosophy: Machine vs Machine

**No longer rely on limited test cases manually written by human developers, but use algorithms, interpreters, and model checking tools to exhaustively attack code boundaries at high dimensions.**

### Absolute Prerequisite

**Rust compiler's safety guarantees only cover memory allocation and basic thread isolation within the safety domain. For complex business invariants, lock-free concurrency logic, and unsafe boundaries, must introduce advanced automated testing defense lines.**

---

## 1. Property-based Testing

### Core Philosophy

**Test "properties" and "invariants", not "specific examples".**

```rust
// ❌ Traditional: Testing specific examples
assert_eq!(add(1, 2), 3);

// ✅ Property-based: Testing general properties
// For all a, b: add(a, b) == add(b, a)
proptest! {
    #[test]
    fn addition_is_commutative(a in any::<i32>(), b in any::<i32>()) {
        prop_assert_eq!(add(a, b), add(b, a));
    }
}
```

### 1.1 Cover Core Algorithms & State Machines

**Specification**: Any state machine involving complex mathematical calculations, custom serialization/deserialization logic, or with clear preconditions/postconditions must use `proptest` for property-based testing.

```rust
// Cargo.toml
[dev-dependencies]
proptest = "1.4"

// tests/property_tests.rs
use proptest::prelude::*;

// ✅ Required: Property tests for serialization
proptest! {
    #[test]
    fn serialize_deserialize_roundtrip(data in any::<MyStruct>()) {
        let encoded = encode(&data);
        let decoded = decode(&encoded).unwrap();
        // Verify: decode(encode(data)) == data
        prop_assert_eq!(data, decoded);
    }
    
    #[test]
    fn state_machine_transitions(initial_state in any::<State>(), 
                                  input in any::<Input>()) {
        let mut state = initial_state;
        let result = state.transition(input);
        
        // Verify invariants hold after transition
        prop_assert!(state.is_valid());
        prop_assert!(result.is_ok() || state.is_error());
    }
}
```

**Verification Loop**: For serialization protocols, must implement property test verification of `decode(encode(data)) == data`.

### 1.2 Strategy Definition & Shrinking

**Specification**: When writing proptest strategies, must precisely define input domain boundaries (such as generating only legal UTF-8 strings or integers in specific ranges) to avoid meaningless inputs causing test failures. When tests fail, proptest will automatically perform "shrinking" to find the minimal input causing failure; must record this minimal input as a regression test case.

```rust
use proptest::{prelude::*, strategy::Strategy};

// ✅ Define precise input domain
fn valid_utf8_string() -> impl Strategy<Value = String> {
    "[a-zA-Z0-9_\\-\\.]+".prop_map(|s| s.to_string())
}

fn user_id() -> impl Strategy<Value = u64> {
    1..=1_000_000u64 // Only valid ID range
}

proptest! {
    #[test]
    fn test_user_creation(
        id in user_id(),
        name in valid_utf8_string(),
        age in 0..=150u8,
    ) {
        let user = User::new(id, name, age)?;
        prop_assert!(user.is_valid());
    }
}
```

**Shrinking Mechanism**: When test fails, proptest automatically finds minimal failing input.

```rust
// Example: Fails when vec length >= 5
proptest! {
    #[test]
    fn test_vec_processing(vec in prop::collection::vec(any::<i32>(), 0..100)) {
        // Fails: [0, 0, 0, 0, 0]
        // Shrunk to minimal failing case automatically
        prop_assert!(process_vec(&vec).is_ok());
    }
}
```

**Record Minimal Failing Case**:
```rust
// Save minimal failing case as regression test
#[test]
fn regression_test_vec_length_5() {
    let vec = vec![0, 0, 0, 0, 0];
    assert!(process_vec(&vec).is_ok());
}
```

### 1.3 Compromise & CI Isolation

**Trade-off**: Property-based testing is extremely time-consuming. Never mix it with regular `cargo test` blocking rapid feedback loops for daily development. Configure via `PROPTEST_CASES` environment variable to run only small numbers locally (e.g., 256 cases), and execute large-scale bombing in nightly CI (e.g., 100,000 cases).

```rust
// tests/property_tests.rs

// Configure test count via environment variable
fn test_count() -> u32 {
    std::env::var("PROPTEST_CASES")
        .unwrap_or_else(|_| "256".to_string())
        .parse()
        .unwrap_or(256)
}

proptest! {
    #![proptest_config(ProptestConfig {
        cases: test_count(),
        .. ProptestConfig::default()
    })]
    
    #[test]
    fn heavy_property_test(data in any::<ComplexType>()) {
        // Local dev: 256 cases (fast)
        // Nightly CI: 100,000 cases (thorough)
        prop_assert!(complex_invariant(&data));
    }
}
```

**CI Configuration**:
```yaml
# .github/workflows/test.yml
jobs:
  test:
    # Fast feedback for PRs
    - run: cargo test
    
  nightly-property-tests:
    # Run daily with massive cases
    - run: PROPTEST_CASES=100000 cargo test --test property_tests
```

**Rule**: Never mix property tests with regular `cargo test` blocking daily development rapid feedback.

---

## 2. Fuzzing

### Core Philosophy

**Introduce genetic mutation and coverage guidance to search for crash-causing malformed data.**

### 2.1 Defense Line for External Inputs

**Specification**: All parsing entry points directly exposed to untrusted external environments (such as HTTP protocol parsers, file format decoders, blockchain transaction validators) must use `cargo fuzz` to write Fuzz Targets.

```bash
# Setup
cargo install cargo-fuzz

# Initialize fuzz target
cargo fuzz init

# Create specific fuzz target
cargo fuzz add parse_http_request
cargo fuzz add decode_image
cargo fuzz add validate_transaction
```

```rust
// fuzz/fuzz_targets/parse_http_request.rs
#![no_main]
use libfuzzer_sys::fuzz_target;
use my_parser::parse_http_request;

fuzz_target!(|data: &[u8]| {
    // Fuzz the parser entry point
    let _ = parse_http_request(data);
});
```

**Protection Goals**:
- ✅ Absolutely no Panic
- ✅ Absolutely no OOM (Out Of Memory)
- ✅ Absolutely no infinite loop hang (Hang)

**Run Fuzz**:
```bash
# Run fuzzer
cargo fuzz run parse_http_request

# With specific timeout (prevent hangs)
cargo fuzz run parse_http_request --timeout=60

# Run with multiple jobs
cargo fuzz run parse_http_request -j8
```

### 2.2 Side-effect-free Target Design

**Specification**: Fuzz Target must be pure computation/parsing logic. Absolutely never include network requests, database writes, or other operations that change external state or cause severe I/O bottlenecks in Fuzz tests.

```rust
// ✅ Correct: Pure computation, no side effects
fuzz_target!(|data: &[u8]| {
    let _ = parse(data);
    let _ = validate(data);
    // No network, no DB, no I/O
});

// ❌ Forbidden: Side effects
fuzz_target!(|data: &[u8]| {
    parse(data)?;
    save_to_database(data); // NO! External state change
    send_network_request(data); // NO! I/O bottleneck
});
```

**Why**: 
- Fuzzing requires millions of iterations
- External I/O will severely slow down fuzzing speed
- Side effects lead to non-reproducible test results

### 2.3 Compromise & Computing Power Allocation

**Trade-off**: Fuzzing is a never-ending, extremely CPU-intensive process. Should not place it in regular PR CI workflows. Standard practice is to integrate with Google OSS-Fuzz or internal dedicated continuous fuzzing clusters for 24x7 non-stop mutation attacks.

```yaml
# .github/workflows/fuzz.yml

# Don't run fuzzing in regular PR CI
# Instead: Run specialized fuzzing cluster 24x7

jobs:
  continuous-fuzzing:
    runs-on: self-hosted-fuzz-cluster
    steps:
      - run: cargo fuzz run parse_http_request -- -max_total_time=86400
      
  oss-fuzz-integration:
    # Integrate with Google OSS-Fuzz
    - run: ./oss-fuzz/build.sh
```

**Rules**:
- ❌ Don't place in regular PR CI workflows
- ✅ Integrate with Google OSS-Fuzz or internal dedicated continuous fuzzing clusters
- ✅ 24x7 non-stop mutation attacks

---

## 3. Concurrency Model Checking

### Core Philosophy

**Take over OS thread scheduling and exhaust all interleaved execution paths**.

Traditional concurrency tests trigger data races by "guessing luck" through sleep or retry loops, which is ineffective in engineering.

### 3.1 Abstract Synchronization Primitive Isolation

**Specification**: Any manually implemented lock-free data structures, or code using complex Atomic*, Condvar for fine-grained thread synchronization, must abstract underlying synchronization primitives.

```rust
// src/sync_wrapper.rs

// Conditional compilation for loom
#[cfg(all(test, loom))]
pub(crate) use loom::sync::atomic::{AtomicUsize, AtomicBool, Ordering};

#[cfg(not(all(test, loom)))]
pub(crate) use std::sync::atomic::{AtomicUsize, AtomicBool, Ordering};

// Use abstracted atomics in production code
pub struct LockFreeQueue<T> {
    head: AtomicUsize,
    tail: AtomicUsize,
    data: Vec<T>,
}

impl<T> LockFreeQueue<T> {
    pub fn push(&self, value: T) {
        // Uses abstracted AtomicUsize
        let tail = self.tail.load(Ordering::Acquire);
        // ... lock-free logic
    }
}
```

### 3.2 Exhaustive Concurrent Execution Tree

**Specification**: Wrap test code using `loom::model` closure. Loom will explore all possible permutations of thread execution order, finding ABA problems or deadlocks that may only occur with one-in-a-million probability.

```rust
// tests/loom_test.rs

#[cfg(loom)]
#[test]
fn test_lock_free_queue_concurrent_push() {
    use loom::thread;
    use crate::sync_wrapper::LockFreeQueue;
    
    loom::model(|| {
        let queue = LockFreeQueue::new();
        
        // Thread 1
        let t1 = thread::spawn({
            let q = queue.clone();
            move || q.push(1)
        });
        
        // Thread 2
        let t2 = thread::spawn({
            let q = queue.clone();
            move || q.push(2)
        });
        
        t1.join().unwrap();
        t2.join().unwrap();
        
        // Verify invariants
        assert_eq!(queue.len(), 2);
    });
}
```

**Loom Explores**:
- All possible thread execution order permutations
- ABA problems (one-in-a-million probability)
- Deadlock scenarios
- Data races

### 3.3 Compromise: Controlling State Explosion

**Trade-off**: As thread count and atomic operation count increase, paths Loom needs to explore grow exponentially (combinatorial explosion). Must compress Loom test scenarios to extreme minimal: test only 2-3 threads, each thread executes only 1-2 conflicting operations. Never run entire system's concurrent business flows in Loom.

```rust
// ✅ Correct: Minimal test scenario
#[cfg(loom)]
#[test]
fn test_minimal_concurrent_scenario() {
    loom::model(|| {
        // Only 2-3 threads
        // Each thread executes 1-2 conflicting operations
        let atomic = AtomicUsize::new(0);
        
        let t1 = thread::spawn(|| atomic.fetch_add(1, Ordering::SeqCst));
        let t2 = thread::spawn(|| atomic.fetch_add(1, Ordering::SeqCst));
        
        t1.join().unwrap();
        t2.join().unwrap();
        
        assert_eq!(atomic.load(Ordering::SeqCst), 2);
    });
}

// ❌ Wrong: Too complex for loom
#[cfg(loom)]
#[test]
fn test_entire_system_concurrent_flow() {
    // DON'T: Run entire business flow with many threads
    // This will cause state explosion
}
```

**Rules**:
- ✅ Test only 2-3 threads
- ✅ Each thread executes only 1-2 conflicting operations
- ❌ Never run entire system's concurrent business flows in Loom

---

## 4. Undefined Behavior Detection

### Core Philosophy

**Before LLVM optimizes your code, use interpreter to precisely capture memory sins**.

### 4.1 Life-or-death Line for Unsafe Code

**Specification**: As long as one line of unsafe code exists in the project (such as calling FFI, using pointer offsets, manual forced type conversions), must forcibly add `cargo miri test` step in CI workflow.

```bash
# Install Miri
rustup component add miri
cargo miri setup

# Run tests with Miri
cargo miri test

# Run specific test with Miri
cargo miri test my_unsafe_function
```

**Capture Targets**:
- ✅ Use-after-free (Dangling pointers)
- ✅ Strict Aliasing Rule Violation
- ✅ Uninitialized memory reads
- ✅ Data races

```rust
// Example: Miri catches this UB
fn dangerous() {
    let ptr = Box::into_raw(Box::new(42));
    unsafe {
        drop(Box::from_raw(ptr));
        println!("{}", *ptr); // UB: Use-after-free
        // Miri will catch this at runtime
    }
}
```

**CI Integration**:
```yaml
# .github/workflows/miri.yml
jobs:
  miri:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Install Miri
        run: |
          rustup toolchain install nightly --component miri
          cargo miri setup
      - name: Run Miri tests
        run: cargo +nightly miri test
```

### 4.2 Shielding Miri-unsupported Operations

**Specification**: Miri is an interpreter and cannot directly execute FFI calls or system calls.

```rust
#[cfg(miri)]
// Mock FFI for Miri
unsafe fn external_c_function() -> i32 {
    42 // Mocked result
}

#[cfg(not(miri))]
// Real FFI for native execution
unsafe fn external_c_function() -> i32 {
    extern "C" { fn real_c_function() -> i32; }
    real_c_function()
}

// Test skips unsupported operations
#[test]
#[cfg_attr(miri, ignore = "Miri doesn't support FFI")]
fn test_with_ffi() {
    unsafe {
        external_c_function();
    }
}
```

### 4.3 Compromise: Trading Execution Efficiency for Correctness

**Trade-off**: Miri's execution speed is typically 50x+ slower than native compilation execution.

**Correct Practice**:
```rust
// ✅ Required: Core unit tests with Miri
#[test]
fn test_unsafe_memory_operation() {
    // Small, focused test
    unsafe {
        // Miri verifies this is safe
        perform_unsafe_operation();
    }
}

// ❌ Avoid: Integration tests with Miri
#[test]
fn test_large_integration_with_unsafe() {
    // Too slow for Miri (50x overhead)
    // Run with native tests instead
}

// ❌ Avoid: Heavy proptest with Miri
proptest! {
    #[test]
    fn heavy_property_test_with_unsafe(data in any::<LargeType>()) {
        // Too slow for Miri
    }
}
```

**Rules**:
- ✅ Only require core unit tests to pass Miri verification
- ❌ Don't use Miri to run high-intensity integration tests
- ❌ Don't use Miri to run large-scale Proptest

---

## Testing Strategy Matrix

| Test Type | Applicable Scenarios | Execution Frequency | Tool |
|-----------|---------------------|-------------------|------|
| Unit Tests | All function logic | Every compilation | `cargo test` |
| Property Tests | Core algorithms, state machines, serialization | Nightly CI | `proptest` |
| Fuzzing | External input parsers | 24x7 continuous | `cargo fuzz` |
| Concurrency Checking | Lock-free data structures, atomic operations | Periodic runs | `loom` |
| UB Detection | All unsafe code | CI mandatory | `cargo miri` |

---

## Performance Checklist

When configuring testing strategies, check in sequence:

### Property-based Testing
- [ ] Core algorithms have property tests
- [ ] Serialization protocols have roundtrip tests
- [ ] Input domain boundaries precisely defined
- [ ] CI isolation (PROPTEST_CASES configuration)

### Fuzzing
- [ ] External parsers have fuzz targets
- [ ] Targets are side-effect-free (pure computation)
- [ ] Integrated with continuous fuzzing cluster
- [ ] Timeout configured to prevent hangs

### Concurrency Model Checking
- [ ] Synchronization primitives abstractly isolated
- [ ] Tests wrapped with loom::model
- [ ] Thread count controlled (2-3 threads)
- [ ] Avoid state explosion

### Undefined Behavior Detection
- [ ] CI mandatory cargo miri test
- [ ] Mock FFI unsupported by Miri
- [ ] Only core unit tests use Miri
- [ ] Avoid large-scale tests with Miri

---

## Related

- [priority-pyramid.md](priority-pyramid.md) — P0 (Safety) verification
- [toolchain.md](toolchain.md) — CI integration for testing
- [performance-tuning.md](performance-tuning.md) — Profiling-guided optimization
