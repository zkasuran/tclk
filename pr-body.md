Fixes #127

## Problem

When multiple competing accept frames target the same public offer in `tclk-offers`, the contract binds to whichever valid accept lands first. Subsequent accepts are rejected with `"accept in status accepted"`.

As documented in #127, during the airdrop measurement period 319 honest agents minted secrets, published statements and were rejected at this gate with no defined path forward. The same rejection message appears for both:
- A replay of the same statement (protocol violation)
- A losing acceptor with a fresh statement (race condition, protocol-compliant)

The specification never stated what the losing acceptor should do.

## What was broken

SPEC §4 described the accept binding without stating the **first-wins** rule explicitly, leaving ambiguity about whether multiple accepts could bind concurrently or whether tie-breaking was implementation-defined.

The rejection at `src/machine.ts:117` is correct behavior, but without specification guidance a losing acceptor had no documented path to clean up its reservation.

## The fix

This commit documents the normative **first-wins** rule in SPEC §4: when competing accepts race, the first valid accept binds the offer and all subsequent accepts are rejected.

Adds application-level guidance: losing acceptors observing that a competing accept has bound the offer SHOULD release their local reservations and uncommitted lock states.

## Changes

- **SPEC.md**: Added explicit first-wins binding rule in §4, plus application guidance for losing acceptors to release reservations
- **tests/tclk.test.ts**: Added comprehensive test suite proving rejection behavior, distinguishability from replay, and no state change on rejection
- **CHANGELOG.md**: Documented under `[Unreleased]`

## Tests

All 107 tests passing (baseline 104 + 3 new):

```
✓ accept: second accept from different agent with different statement is rejected
✓ accept: multiple losing acceptors each receive the same rejection  
✓ accept: rejection is distinguishable from replay by inspecting the statement
```

Wire format unchanged. No breaking changes.

## Related

Per @sv's comment on #127, this addresses the losing-acceptor semantics. PR #141 implements related wording changes against #140.

---

**AI disclosure**: AI assistance (Claude, Anthropic) was used in developing this change. The design, review and verification were done by the author. Verified locally before submitting: all tests passing, no lint errors.
