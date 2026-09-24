# unit-test-antislop

## Install

Paste this into Claude Code:

```
Install the skill from https://github.com/danihrndzld/unit-test-antislop: clone it into ~/.claude/skills/unit-test-antislop (git clone https://github.com/danihrndzld/unit-test-antislop ~/.claude/skills/unit-test-antislop), confirm SKILL.md is there, and from now on load the unit-test-antislop skill (and tell any subagent to load it) whenever writing, reviewing, or extending unit tests or doing TDD.
```

Or manually:

```sh
git clone https://github.com/danihrndzld/unit-test-antislop ~/.claude/skills/unit-test-antislop
```

Project-only install: clone into `.claude/skills/unit-test-antislop` inside the repo instead.

## What it does

A Claude Code skill for subagents that write unit tests. It enforces:

- **TDD from functional requirements**: list FRs first, one test per FR, see every test fail before it passes.
- **AAA structure**, one Act, no control flow in tests.
- **Observable behavior only**: no asserting on private internals or stub calls.
- **Stubs for incoming, mocks for outgoing** dependencies, one mock per test.
- **Fast and deterministic**: no network, disk, real DB, clock, or randomness.
- **Fact-based names**: `order_with_past_delivery_date_is_rejected`, not `test_validate_should_work`.
- **Factories over `beforeEach`**, hardcoded expected values, no tautologies.
- **No tests for trivial code**: getters, DTOs, pass-throughs, framework behavior.

It ends with a slop checklist and a fixed report format (FR → test traceability, red-seen, run results) so orchestrators get signal back, not prose.

See [SKILL.md](SKILL.md).
