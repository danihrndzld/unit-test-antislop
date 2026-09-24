---
name: unit-test-antislop
description: Use when writing, reviewing, or extending unit tests, or doing TDD. Every test must trace to a functional requirement, fail first for the right reason, assert observable behavior, and earn its maintenance cost. Rejects tautologies, over-mocking, trivial-code tests, and coverage padding.
---

# Unit Test Anti-Slop

A test exists to fail when a requirement breaks and to stay green when the code is refactored. A test that does neither is noise. Delete noise.

## 0. Start from functional requirements

TDD is driven by **functional requirements (FRs)**, not by the code that exists and not by coverage numbers.

Before writing a test, write the FR list for the unit, one line each:

```
FR-1  An order with a past delivery date is rejected.
FR-2  Orders over Q10,000 require manager approval.
FR-3  A rejected order sends no confirmation email.
```

Sources, in order: the ticket/spec/issue, docs or README, the caller's expectations, the existing behavior (only if nothing else exists — and say so). If an FR is ambiguous, write down your assumption next to it; do not invent requirements.

Rules:
- Every test maps to exactly one FR. Put the FR id in the test name or a one-line comment.
- An FR with no test is a gap. A test with no FR is a candidate for deletion.
- For each FR, cover: the happy path, each boundary (off-by-one on both sides), and each distinct failure mode the FR implies. Nothing else.

## 1. Red → Green → Refactor

1. **Red.** Write one test for one FR. Run it. It must fail, **and the failure message must be the one you expect** (assertion failure, not `ImportError`/`NameError`/typo). A test you never saw fail proves nothing.
2. **Green.** Write the minimum production code that passes. No speculative branches, params, or config (YAGNI).
3. **Refactor.** Clean production and test code with all tests green. Tests must not change during a pure refactor; if they must, they were coupled to implementation (see §3).

Adding tests to existing code (no TDD possible): after the test passes, break the production line it covers (flip a condition, change a constant), confirm the test goes red, then restore. If it stays green, the test is worthless — fix or delete it.

## 2. Structure: Arrange, Act, Assert

```python
def test_order_with_past_delivery_date_is_rejected():  # FR-1
    order = make_order(delivery_date=date(2020, 1, 1))   # Arrange

    result = order.validate(today=date(2026, 9, 24))     # Act

    assert result.errors == ["delivery_date_in_past"]    # Assert
```

- **One Act** per test, ideally one line. Two Act/Assert cycles = two tests.
- **No control flow** in tests: no `if`, `switch`, `for`, `while`, `try/except` for flow. Use the framework's parametrization for tables of inputs (`pytest.mark.parametrize`, `it.each`, `[Theory]`).
- Blank lines separate the three phases. `# Arrange/# Act/# Assert` comments are optional; don't add them if the blank lines make it obvious.

## 3. Assert observable behavior, not implementation

Assert on: return values, resulting public state, errors raised, and messages sent across a system boundary.

Never assert on: private methods/fields, call order of internal collaborators, number of times an internal helper ran, intermediate data structures.

Test: "If I rewrite the internals with the same behavior, does this test still pass?" If no, rewrite the test.

## 4. Stubs in, mocks out

| Dependency | Double | Assert on it? |
|---|---|---|
| **Incoming** (data the SUT reads: repo, clock, config, API response) | Stub / fake | **Never.** Asserting a stub was called is overspecification. |
| **Outgoing, unmanaged** (side effects others observe: email, message bus, payment gateway, 3rd-party API) | Mock | Yes — **one mocked interaction per test.** |
| **Managed / in-process** (your own classes, your own DB used only by your app) | Real object | Assert on resulting state. |

- Mock only at the true edge of the system. Mocking your own classes couples tests to structure.
- Don't mock what you don't own directly — wrap it in a thin adapter and mock the adapter.
- More than one mock verification in a test = multiple requirements = split it.

## 5. Fast, isolated, deterministic

- No real network, filesystem, database, or external process in a unit test. If the logic needs them, extract the logic (Humble Object) and test it in memory; leave the thin I/O shell for integration tests.
- Inject time, randomness, UUIDs, and env. Never call `now()`, `random()`, or read env vars inside the tested logic without an injection seam.
- No shared mutable state between tests, no dependence on execution order. Each test must pass alone and in random order.
- No `sleep`. No retries. A flaky unit test is a bug — fix the nondeterminism, don't rerun.
- Target: the whole unit suite runs in seconds.

## 6. Names state facts

Name = the FR as a statement of fact, in domain terms. Someone reading only the failure list must know what broke.

- Good: `order_with_past_delivery_date_is_rejected`, `transfer_over_daily_limit_requires_approval`
- Bad: `test_validate`, `test_1`, `test_order_should_work`, `testValidateReturnsFalseWhenDateLessThanNow`

No "should". No method names as the whole name. No implementation words (`returns_false`, `calls_repo`).

## 7. Factories over shared setup

- Build fixtures with small factory functions / test data builders with sensible defaults; override only what the test cares about: `make_order(total=10_001)`.
- The values that matter to the test must be **visible in the test**. Anything hidden in `beforeEach`/`setUp`/class fixtures that affects the outcome is a readability bug.
- Shared setup is allowed only for things irrelevant to every assertion (e.g., constructing a stub clock).

## 8. What to test (and what not to)

Test, in priority order:
1. Domain rules and decisions (validation, pricing, permissions, state transitions).
2. Algorithms and non-trivial transformations (parsers, calculators, mappers with logic).
3. Boundaries and error paths named by an FR.

Do not test:
- Getters/setters, auto-properties, constructors that only assign.
- Pass-through methods with no logic.
- Framework or library behavior (the ORM saves, the router routes, `json.dumps` works).
- Type definitions, constants, DTOs.
- Controllers/handlers that only wire things — that's integration-test territory.

Unit tests are the broad base of the pyramid. If a behavior can only be verified end-to-end, don't fake it with a mock-heavy "unit" test; flag it for integration/E2E.

## 9. No logic, no tautology

- Expected values are **hardcoded literals** derived independently (by hand, from the spec, from a domain expert). Never computed in the test.
- Never re-implement the production formula in the test.

```python
# Tautology — passes even if the formula is wrong
assert tax(100) == 100 * TAX_RATE

# Signal
assert tax(Decimal("100.00")) == Decimal("12.00")
```

- No string building, loops, or math to construct expectations.
- No snapshot tests of large outputs as a substitute for stating what matters.

## Slop checklist — reject the test if any is true

- [ ] It has no FR it traces to.
- [ ] You never saw it fail (or broke the code and it stayed green).
- [ ] It asserts `is not None`, `toBeDefined()`, `toBeTruthy()`, or type-only checks as the main assertion.
- [ ] It asserts that a stub was called.
- [ ] It verifies more than one mock, or has more than one Act.
- [ ] It contains `if`/loops/try-except, or computes its expected value.
- [ ] It tests a getter, setter, constructor, DTO, or framework behavior.
- [ ] It would break under a behavior-preserving refactor.
- [ ] It touches network, disk, real DB, real clock, or randomness.
- [ ] Its name contains "should", or is a method name, or needs the body to be understood.
- [ ] It duplicates another test's FR + input class with a different literal (same equivalence class, no new signal).
- [ ] It exists to raise coverage.

Fewer, sharper tests beat many shallow ones. If asked for "more coverage", add tests only for uncovered FRs, boundaries, or failure modes — never for lines.

## Report back (when running as a subagent)

Return exactly:

```
FRs:        FR-1 … FR-n (source: <ticket/spec/assumed>)
Tests:      <n> added, <n> changed, <n> deleted (with reason per deletion)
Traceability: FR-1 → test_a, test_b; FR-2 → test_c; FR-3 → GAP (<why>)
Red seen:   yes for all | list exceptions
Run:        <command> → <passed>/<total>, <duration>
Assumptions: <each assumed requirement, one line>
Out of scope: <behaviors needing integration/E2E>
```

No summaries of what tests "ensure" or "validate". Facts only.
