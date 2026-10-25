# Appendix: C++ and Kudu checks from the Kudu Reviewer's reviews

Use this appendix together with the main profile only when the change is C++ (and especially Apache
Kudu). It lists the language- and codebase-specific checks that he applied on top of his general
reasoning.


### Status and errors
- Use `RETURN_NOT_OK`, `RETURN_NOT_OK_PREPEND(s, "context")` to add context, `WARN_NOT_OK` for
  errors that are logged and ignored, and `ignore_result()` for results that are dropped on
  purpose. Report unchecked `Status` returns.
- Build a `Status` from an RPC error with `StatusFromPB(resp.error().status())`, the way the
  rest of the code does.
- Calls that can do I/O must not be wrapped in `CHECK_OK`; return the error instead.
- `CHECK` versus `DCHECK` versus `LOG(DFATAL)`: invariants that are cheap to check and must hold
  get `DCHECK`; conditions that indicate a programming error on a cold path can `CHECK`; a library
  should not crash on an I/O error. `DCHECK_NE(ptr, nullptr)` gives a clearer message than
  `DCHECK(ptr)`.
- Preserve a `Status` through an OUT parameter rather than losing it in a factory return.

### Ownership and C++ idiom
- `unique_ptr` for new owning pointers; ask why `shared_ptr` or `scoped_refptr` is needed.
  Use `make_shared.h` helpers for protected constructors.
- Pass sink parameters by value and `std::move` them; pass read-only objects by const reference
  ("Not cref?"); mark accessors `const`; question `mutable`.
- Annotate literal arguments: `Foo(/*force=*/true)`.
- `enum class` over plain enums, `static constexpr` (not `constexpr static`), `#pragma once`,
  `override`, `FALLTHROUGH_INTENDED` for a deliberate fallthrough, `emplace_back`, range-for,
  C++ headers (`<cstdint>`), `using` declarations for common std types instead of repeated `std::`.
- Formatting follows the Google C++ style: continuation lines are aligned with the opening
  parenthesis or indented four spaces, and alignment errors on wrapped arguments and conditions
  are worth a nit; `Type* x` and `Type** x`, not `Type *x`; binary operators such as `&&` end the
  line rather than start the next one; functions and logical sections are separated by blank
  lines; `CHECK_EQ(expected, actual)` puts the expected value first. Prefer an explicit type over
  `auto` for a plain integer.
- Includes must be sorted, follow IWYU, and avoid unneeded headers; prefer forward declarations
  for types held by pointer. CMake source lists stay sorted, and unused dependencies are removed.
- Use map-util helpers (`FindOrNull`, `FindCopy`, `InsertOrDie`, `EmplaceOrDie`) instead of
  `find()` against `end()`.
- Use `PREDICT_FALSE` on unlikely error branches, but not where it would go stale if a flag default
  changes.
- Avoid compiler-specific extensions and hardcoded ISA flags (`-mavx2`); dispatch on CPU features
  at runtime.

### Kudu infrastructure
- Use `kudu::Thread` (visible in /threadz and metrics) rather than `std::thread`, Kudu
  synchronization primitives with `MonoDelta` timeouts, `MonoTime` for elapsed time,
  `ThreadPool`/`ThreadPoolToken` rather than ad hoc threads, the maintenance manager for
  background work, `PeriodicTimer` for periodic tasks, and `ScopedCleanup`/`MakeScopedCleanup`
  for cleanup that must run on every return path (cancel it on success).
- Use `Env` and `RWFile` abstractions instead of raw syscalls, including in tests.
- `SecureDebugString()` for logging protobufs. Put `s.ToString()` at the end of a log line.
- Protobuf: new fields are `optional`; reason about the default value that an old peer reads;
  do not send a sentinel when the field can be omitted.
- Methods that only tests call get a `ForTests` suffix.

### gflags
- New flags get tags (`advanced`, `experimental`, `evolving`, `unsafe`, `hidden` for test-only
  flags) and a description that states units and incompatibilities.
- Cross-flag constraints use `GROUP_FLAG_VALIDATOR`; single-flag checks use a validator.
- Declare flags with `DECLARE_*` at the top of the file and read `FLAGS_x` directly instead of
  looking up the value by string; when a program needs a different default, change the default
  (so that the command line still overrides it) rather than the value.
- Ask whether a per-request parameter fits better than a new server-wide flag.

### Metrics
- New metrics need a deliberate level (`kDebug`, `kInfo`, `kWarn`) and a description that says
  what happens before the first event and after a restart. A numeric metric must never carry a
  string like "N/A"; hide or omit it instead. Use `FunctionGauge` for computed values and a
  monotonic clock for durations. Unpublish metric entities that go away.

### Tests
- Use `ASSERT_*`/`EXPECT_*` in tests, not `CHECK_*`, on the main test thread. Use `ASSERT_OK`
  on every `Status` in a test.
- Helpers that assert return `void` and are called through `NO_FATALS(...)`; otherwise failures
  surface only at the end of the test.
- Overridden `SetUp()` calls the parent first; `TearDown()` calls it last.
- Use `ASSERT_EVENTUALLY` (or `AssertEventually`) instead of sleeps for asynchronous effects, and
  keep only the polled condition inside the block.
- Do not mix internal and external mini-cluster access in one test. Use the existing fixtures
  (`KuduTest`, `ExternalMiniClusterITestBase`, `SeedRandom()`, `env_`) and
  `OverrideFlagForSlowTests`/`AllowSlowTests()` only when the test is really slow.
- New tool actions need coverage in `kudu-tool-test`, including the help output.

### Build and scripts
- New scripts get the license header and a short description of what they do. Shell scripts
  quote expansions, use `$(...)`, and validate their inputs upfront.
- Third-party patches are named after the upstream issue, and a patch that upstream will never
  accept needs a strong reason.
