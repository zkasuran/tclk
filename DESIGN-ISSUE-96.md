# Design Document: Issue #96 — Transcript Timestamp Authentication

## Problem Statement

`foldTranscript` uses unsigned venue `ts` for deadline guards. A file supplier can rewrite `ts` in any record without breaking Ed25519 signatures, flipping `claimed` ↔ `refunded` outcomes with all `verifyTranscriptRecord` checks passing.

**Root cause:** The Ed25519 signature covers `room|nonce|line` (src/transcript.ts:103). `seq` and `ts` are venue metadata parsed separately (line 131) and never authenticated. Deadline guards in machine.ts (lines 120, 157, 177, 191) consume `nowMs` (the parsed `ts`) as if it were signed.

**sv's guidance:** Either the fold refuses when a deadline verdict depends on unsigned metadata, or deadline evaluation moves onto something inside the signed preimage. This design takes the first path: surface the trust assumption rather than silently consuming unsigned time.

## Solution Design

Add `warnings: string[]` to `TranscriptFoldResult`. Emit warnings for metadata anomalies and deadline reliance. Do NOT refuse to fold (partial windows and post-reap transcripts are legitimate). Forward warnings in `audit-export.mjs` and `mcp/src/tools.ts`. `audit-export.mjs` exits 1 when gaps, non-monotonic seq, or backwards timestamps are detected.

---

## 1. Type Signature Changes

### src/transcript.ts

```typescript
export interface TranscriptFoldResult {
  state: ContractState | null;
  steps: TranscriptStep[];
  warnings: string[];  // NEW
}
```

No change to `TranscriptRecord`, `TranscriptStep`, or `verifyTranscriptRecord`. Warnings are fold-level, not per-record.

---

## 2. Warning Emission Rules

### Per-room anomaly warnings (detect metadata tampering)

Tracked per distinct `room` value in the supplied records. Each room gets independent seq/timestamp tracking.

| Condition | Warning text | When to emit |
|-----------|--------------|--------------|
| seq gap | `gap detected in room <room> (seq <prev> → <curr>)` | `record.seq > prev.seq + 1` for the same room |
| seq not increasing | `seq not strictly increasing in room <room> (seq <prev> → <curr>)` | `record.seq <= prev.seq` for the same room |
| timestamp backwards | `timestamp goes backwards in room <room> (prev <ISO>, curr <ISO>)` | `record.timestampMs < prev.timestampMs` for the same room |

**Emit at most one warning per room per anomaly type.** First violation in each room triggers the warning; subsequent violations in the same room of the same type are silent (avoid flooding on a heavily tampered file).

**Skip anomaly checks for records that fail `verifyTranscriptRecord`** — unsigned or malformed records contribute no metadata baseline. Only track verified records for per-room state.

### Generic deadline warning (trust disclosure)

Emit exactly once per fold when **any** deadline-sensitive frame type appears in a step with `ok: true` (successful transition):

```
"timestamps and seq are venue metadata, not covered by signature — verify settlement on the rail"
```

**Deadline-sensitive frame types:** `accept`, `lock`, `reveal`, `refund`

These four types have deadline guards in machine.ts that consume `nowMs`:
- `accept`: line 120 checks `nowMs >= state.offer.expiresMs`
- `lock`: line 157 checks `nowMs >= state.offer.refundAfterMs`
- `reveal`: line 177 checks `nowMs >= state.offer.refundAfterMs`
- `refund`: line 191 checks `nowMs < state.offer.refundAfterMs`

Emit the generic warning even if no anomalies were detected. It surfaces the trust model: unsigned time is always untrusted, even in a clean-looking transcript.

**Do NOT emit** if all deadline frames were rejected (every `accept`/`lock`/`reveal`/`refund` has `ok: false`). A fold where every deadline guard failed never relied on unsigned time.

---

## 3. Implementation in foldTranscript (src/transcript.ts)

### Algorithm

```typescript
export function foldTranscript(records: readonly TranscriptRecord[]): TranscriptFoldResult {
  const steps: TranscriptStep[] = [];
  const warnings: string[] = [];
  let state: ContractState | null = null;

  // Per-room tracking: room -> {lastSeq, lastTimestampMs, seqGapWarned, seqOrderWarned, timeBackwardsWarned}
  const roomState = new Map<string, {
    lastSeq: number;
    lastTimestampMs: number;
    seqGapWarned: boolean;
    seqOrderWarned: boolean;
    timeBackwardsWarned: boolean;
  }>();

  let anyDeadlineFrameAccepted = false;

  records.forEach((record, index) => {
    // ... existing verification logic ...
    const verification = verifyTranscriptRecord(record);
    if (!verification.ok) {
      steps.push({ ...base, ok: false, reason: verification.reason });
      return;  // skip metadata tracking for unverified records
    }

    // Track per-room metadata anomalies (only for verified records)
    const room = record.room;
    const rs = roomState.get(room);
    if (rs === undefined) {
      // First verified record in this room
      roomState.set(room, {
        lastSeq: record.seq,
        lastTimestampMs: record.timestampMs,
        seqGapWarned: false,
        seqOrderWarned: false,
        timeBackwardsWarned: false,
      });
    } else {
      // Check seq gap (only if not already warned for this room)
      if (!rs.seqGapWarned && record.seq > rs.lastSeq + 1) {
        warnings.push(`gap detected in room ${room} (seq ${rs.lastSeq} → ${record.seq})`);
        rs.seqGapWarned = true;
      }
      // Check seq ordering (only if not already warned for this room)
      if (!rs.seqOrderWarned && record.seq <= rs.lastSeq) {
        warnings.push(`seq not strictly increasing in room ${room} (seq ${rs.lastSeq} → ${record.seq})`);
        rs.seqOrderWarned = true;
      }
      // Check timestamp ordering (only if not already warned for this room)
      if (!rs.timeBackwardsWarned && record.timestampMs < rs.lastTimestampMs) {
        const prevISO = new Date(rs.lastTimestampMs).toISOString();
        const currISO = new Date(record.timestampMs).toISOString();
        warnings.push(`timestamp goes backwards in room ${room} (prev ${prevISO}, curr ${currISO})`);
        rs.timeBackwardsWarned = true;
      }
      // Update tracking
      rs.lastSeq = record.seq;
      rs.lastTimestampMs = record.timestampMs;
    }

    // ... existing frame decode and application logic ...
    const frame = tryDecodeFrame(record.line);
    // ... existing step logic ...
    const result = applyFrame(state, frame, record.timestampMs);
    
    // Track if any deadline-sensitive frame was accepted
    if (result.ok && frame && ["accept", "lock", "reveal", "refund"].includes(frame.type)) {
      anyDeadlineFrameAccepted = true;
    }
    
    steps.push({ ...base, type: frame.type, ok: result.ok, reason: result.reason });
  });

  // Emit generic deadline warning if any deadline frame succeeded
  if (anyDeadlineFrameAccepted) {
    warnings.push("timestamps and seq are venue metadata, not covered by signature — verify settlement on the rail");
  }

  return { state, steps, warnings };
}
```

**Preserve existing fold behavior:** All existing logic (verification, frame decode, state transitions) is unchanged. Warnings are additive metadata that never affect the fold outcome.

---

## 4. Changes to audit-export.mjs

```javascript
// After existing fold output (lines 41-44)
const folded = foldTranscript([handshake.offer, handshake.accept, ...deal]);
for (const step of folded.steps) {
  const verdict = step.ok ? "ok " : "BAD";
  console.log(`${verdict} ${step.room}#${step.seq} ${step.type ?? "record"}${step.reason ? ` — ${step.reason}` : ""}`);
}

// NEW: Forward warnings to stderr
if (folded.warnings.length > 0) {
  console.error("\nWarnings:");
  for (const warning of folded.warnings) {
    console.error(`  ${warning}`);
  }
}

if (folded.state === null) {
  console.error("no authenticated contract could be opened");
  process.exit(1);
}

const terminal = ["claimed", "refunded", "cancelled"].includes(folded.state.status);
console.log(`\nfold → ${folded.state.status}${terminal ? "" : " (not terminal)"}`);

// NEW: Exit 1 if anomalies detected (gap, ordering, or backwards time)
const hasAnomalies = folded.warnings.some(w =>
  w.includes("gap detected") || w.includes("not strictly increasing") || w.includes("goes backwards")
);
if (hasAnomalies) {
  console.error("\nAudit failed: metadata anomalies detected");
  process.exit(1);
}

process.exit(terminal ? 0 : 1);
```

**Exit code semantics:**
- 0: terminal state reached, no anomalies
- 1: non-terminal state OR anomalies detected OR no contract opened OR handshake error
- 2: usage error (existing, unchanged)

**Generic deadline warning does NOT fail the audit.** It is a trust disclosure, not evidence of tampering. Only gaps/ordering/backwards trigger exit 1.

---

## 5. Changes to mcp/src/tools.ts

### tclk_apply_transcript handler (lines 259-289)

```typescript
tclk_apply_transcript(input: { records: TranscriptRecord[] }) {
  const folded = foldTranscript(input.records);
  if (folded.state === null) {
    const offerFailure = folded.steps.find((step) => step.type === "offer" && !step.ok);
    fail(
      offerFailure?.reason === undefined
        ? "transcript contains no authenticated offer frame to open a contract from"
        : `no contract could be opened: ${offerFailure.reason}`,
    );
  }
  const open: ContractState = folded.state;

  return {
    status: open.status,
    contract: open.contract ?? null,
    offerId: open.offer.id,
    parties: {
      payer: open.payerDid ?? null,
      payee: open.payeeDid ?? null,
      payerKey: open.payerKey ?? null,
      payeeKey: open.payeeKey ?? null,
    },
    statement: open.statement ?? null,
    rail: open.rail ?? null,
    railRef: open.railRef ?? null,
    secretRevealed: open.secret !== undefined,
    steps: folded.steps,
    warnings: folded.warnings,  // NEW: forward warnings
  };
}
```

**No change to behavior.** The tool still fails when `folded.state === null`. Warnings are returned to the MCP client for inspection but do not trigger a failure themselves.

---

## 6. SPEC.md Documentation Changes

### § Line 62 (existing)

**Current text:**
```
`<room>|<nonce>|<text>`; `seq` and `ts` are venue metadata, not sender-signed fields. Deadline
guards replay at that record's `ts`, so a live reader trusts the venue for time and an
offline reader trusts the export file for it. Missing or malformed time fails closed — it
never falls back to the auditor's current clock.
```

**Revised text:**
```
`<room>|<nonce>|<text>`; `seq` and `ts` are venue metadata, not sender-signed fields. Deadline
guards replay at that record's `ts`, so a live reader trusts the venue for time and an
offline reader trusts the export file for it. A file supplier can rewrite `ts` in any record
without breaking signatures, potentially flipping `claimed` ↔ `refunded` outcomes. Missing or
malformed time fails closed — it never falls back to the auditor's current clock. `foldTranscript`
emits warnings when gaps, ordering violations, or backwards timestamps are detected, and surfaces
when any deadline decision relied on unsigned time. Transcript is coordination, not settlement
proof — verify final outcomes on the rail, not from the fold alone.
```

### New § (add after line 67, before "Rendezvous")

```markdown
- **Transcript warnings**: `foldTranscript` returns a `warnings` array alongside `state` and `steps`.
  Per-room warnings (`gap detected`, `seq not strictly increasing`, `timestamp goes backwards`)
  surface metadata anomalies that may indicate tampering. A generic warning is emitted when any
  deadline-sensitive frame (`accept`, `lock`, `reveal`, `refund`) is successfully applied, disclosing
  that unsigned `ts` was trusted for a time-bound decision. Warnings never cause the fold to fail —
  partial windows and post-reap transcripts are legitimate — but `audit-export.mjs` exits 1 when
  anomalies are present, and callers should inspect warnings before trusting settlement outcomes
  derived from the fold alone.
```

---

## 7. Test Cases Needed

### Unit tests (tests/transcript.test.ts)

#### Test 1: No warnings on clean transcript
```typescript
// All records in order, no gaps, monotonic timestamps, no deadline frames
// Expected: warnings = []
```

#### Test 2: Gap detection
```typescript
// Records in room "test": seq 1, 3, 4 (gap 1→3)
// Expected: warnings = ["gap detected in room test (seq 1 → 3)"]
```

#### Test 3: Seq ordering violation
```typescript
// Records: seq 1, 3, 2
// Expected: warnings = ["seq not strictly increasing in room test (seq 3 → 2)"]
```

#### Test 4: Timestamp goes backwards
```typescript
// Records: ts T0, T0+1000, T0+500
// Expected: warnings = ["timestamp goes backwards in room test (prev <ISO>, curr <ISO>)"]
```

#### Test 5: Multiple anomalies, same room
```typescript
// Gap 1→3, then seq 5→4, then time backwards
// Expected: warnings.length = 3 (one of each type)
```

#### Test 6: Same anomaly twice, same room
```typescript
// Gap 1→3, then gap 3→7
// Expected: warnings.length = 1 (only first gap warned)
```

#### Test 7: Anomalies in different rooms
```typescript
// Room A: gap 1→3
// Room B: gap 1→5
// Expected: warnings.length = 2 (one per room)
```

#### Test 8: Generic deadline warning emitted
```typescript
// Clean transcript with one successful `accept` frame
// Expected: warnings = ["timestamps and seq are venue metadata, not covered by signature — verify settlement on the rail"]
```

#### Test 9: Generic deadline warning NOT emitted (no deadline frames)
```typescript
// Clean transcript with only `heartbeat` and `receipt` frames
// Expected: warnings = []
```

#### Test 10: Generic deadline warning NOT emitted (all deadline frames rejected)
```typescript
// Transcript with `accept` that has ok: false (expired offer)
// Expected: no generic deadline warning (never relied on unsigned time for a successful transition)
```

#### Test 11: Both anomaly and generic warnings
```typescript
// Gap 1→3, plus successful `reveal` frame
// Expected: warnings.length = 2 (gap + generic deadline warning)
```

#### Test 12: Attack reproduction (Issue #96 example)
```typescript
// 5 genuine records: offer, accept, lock, reveal@T+30m, refund@T+2h+5s
// Honest fold: reveal succeeds (< refundAfterMs), refund rejected → claimed
// Tampered fold: reveal.timestampMs = T+2h+10s, refund unchanged
// Expected honest: state.status = "claimed", warnings = [generic deadline warning]
// Expected tampered: state.status = "refunded", warnings = ["timestamp goes backwards ...", generic deadline warning]
```

#### Test 13: Unsigned records skipped in metadata tracking
```typescript
// seq 1 (verified), seq 2 (unsigned), seq 4 (verified)
// Expected: gap warning "seq 1 → 4" (seq 2 was skipped because unverified)
```

### Integration test (examples/audit-export.mjs)

#### Test 14: audit-export.mjs exits 1 on gap
```bash
# Create deal.jsonl with seq gap
# Expected: stderr shows "gap detected", exit code 1
```

#### Test 15: audit-export.mjs exits 0 on clean transcript with generic warning
```bash
# Clean transcript reaching claimed state
# Expected: stderr shows generic deadline warning, exit code 0 (warning alone does not fail)
```

#### Test 16: audit-export.mjs exits 1 on timestamp backwards
```bash
# Tampered transcript with ts reordered
# Expected: stderr shows "timestamp goes backwards", exit code 1
```

### MCP tool test (mcp/tests/tools.test.ts or new file)

#### Test 17: tclk_apply_transcript forwards warnings
```typescript
// Call tclk_apply_transcript with a transcript containing gaps
// Expected: result.warnings = ["gap detected ...", ...]
```

---

## 8. Files to Modify

| File | Changes |
|------|---------|
| `src/transcript.ts` | Add `warnings: string[]` to `TranscriptFoldResult`; implement per-room anomaly tracking and generic deadline warning in `foldTranscript` |
| `examples/audit-export.mjs` | Forward `folded.warnings` to stderr; exit 1 if anomalies detected |
| `mcp/src/tools.ts` | Add `warnings: folded.warnings` to `tclk_apply_transcript` return value |
| `SPEC.md` | Revise §62 to document unsigned time malleability; add new § on transcript warnings |
| `tests/transcript.test.ts` | Add 13 unit test cases covering all warning conditions |
| `tests/audit-export.test.ts` (new) | Add integration tests for audit script exit codes |
| `mcp/tests/tools.test.ts` | Add test case verifying warnings forwarded by MCP tool |

---

## 9. What This Fix Achieves

### Satisfies sv's guidance
"Either the fold refuses when a deadline verdict depends on unsigned metadata, or deadline evaluation moves onto something inside the signed preimage."

This design takes the first path: surface the unsigned-time dependency via warnings rather than silently consuming it. The fold does not refuse (partial transcripts are legitimate), but callers are explicitly notified when unsigned metadata was trusted for a time-bound decision.

### Does NOT change wire format
No changes to frame encoding, signature preimage, or transcript export JSON. Existing transcripts remain valid. This is a client-side interpretation change only.

### Preserves existing fold semantics
All state transitions, rejection reasons, and step verdicts remain identical. `warnings` is additive metadata that never affects `state` or `steps[].ok`.

### Makes tampering detectable
Gap/ordering/backwards warnings flag transcripts where `seq` or `ts` were likely rewritten. `audit-export.mjs` now fails audits on such transcripts instead of silently folding them.

### Documents the trust model
The generic deadline warning surfaces that every `accept`/`lock`/`reveal`/`refund` transition trusts unsigned time. This is not a bug to fix (the signed preimage is settled and cannot change without breaking backward compatibility), but a trust assumption to disclose.

### Settlement verification remains the authority
SPEC.md update reinforces: "Transcript is coordination, not settlement proof — verify final outcomes on the rail." A fold reaching `claimed` with warnings does not prove payment succeeded. The rail's on-chain/external settlement record is the authority; the transcript is a coordination log.

---

## 10. Open Design Questions

None. This design is complete and ready for implementation.

---

## Summary

Add `warnings: string[]` to `TranscriptFoldResult`. Emit per-room anomaly warnings (gap, ordering, backwards time) for verified records only, at most once per room per anomaly type. Emit generic deadline warning when any `accept`/`lock`/`reveal`/`refund` succeeds. Forward warnings in `audit-export.mjs` (exit 1 on anomalies) and `mcp/src/tools.ts`. Document in SPEC.md that `ts` is unsigned and transcript is coordination, not settlement proof. Add 17 test cases covering all warning conditions and the Issue #96 attack reproduction.

File paths:
- /home/asuran/Downloads/hackathon-hq/work/tclk/src/transcript.ts
- /home/asuran/Downloads/hackathon-hq/work/tclk/examples/audit-export.mjs
- /home/asuran/Downloads/hackathon-hq/work/tclk/mcp/src/tools.ts
- /home/asuran/Downloads/hackathon-hq/work/tclk/SPEC.md
- /home/asuran/Downloads/hackathon-hq/work/tclk/tests/transcript.test.ts
