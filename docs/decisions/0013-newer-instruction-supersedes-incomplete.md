# 0013: A newer instruction supersedes an incomplete one

- Status: Accepted (Buzyrun fork)
- Date: 2026-11-20

## Context

The reassembler retains one incomplete instruction identifier. Until this
decision, a fragment with another identifier was rejected with
`TooManyIncompleteInstructions`, and the Session driver mapped that rejection
to the fatal `SessionError::ResourceLimit`.

On a lossy path this ends healthy Sessions. Stock `mosh-server` 1.4.0 splits a
full-screen update into several fragments of up to 1,252 datagram bytes. When
one fragment of such an instruction is lost, the instruction stays incomplete,
and the next multi-fragment instruction from the server ends the Session.

Black-box observation (2026-11-20, Outpost plan 12-A, macOS, Homebrew
`mosh-server` 1.4.0, loopback relay that dropped datagrams in both
directions with 100 ms delay):

| Loss | Sessions that ended with `ResourceLimit` |
| --- | --- |
| 15% | 2 of 2 |
| 20% | 2 of 2 (1 of 2 without delay) |
| 30% | 2 of 2 |

The same runs with the reassembler change below kept every Session alive at
15% and 30% loss.

Rejecting only the newer fragment, as `docs/limits.md` described, would not
end the Session, but the retained instruction would block every later
multi-fragment update for up to the 10-second incomplete lifetime.

## Decision

- Fragment identifiers from one sender only grow. When a fragment arrives for
  an identifier newer than the retained incomplete instruction, the
  reassembler drops the retained instruction and starts the newer one.
- A fragment for an identifier older than the retained instruction is dropped
  as `ReassemblyOutcome::Stale` without changing retained state. It is not a
  Session error.
- The one-incomplete-instruction, 749-fragment, 1 MiB, and 10-second bounds are
  unchanged. A newer fragment whose body alone exceeds the 1 MiB instruction
  bound is rejected before the retained instruction is dropped.
- `ReassemblyError::TooManyIncompleteInstructions` is removed.

## Assumption

This decision assumes that fragment identifiers from one sender only grow, as
the encoder in this crate does. The local stock fixtures have not yet recorded
stock identifier generation (`docs/protocol-contract.md` keeps it outside the
contract). The failure mode is bounded: if a stock server ever reused or
lowered an identifier, the newer fragments would be dropped as `Stale` until
the retained instruction expires after 10 seconds. The Session would not end.
Record identifier evidence in `docs/provenance.md` when a fixture captures it.

## Consequences

- Loss of one fragment no longer ends a Session or stalls later updates. The
  sender's next state difference repairs the screen, as for any lost
  single-fragment datagram.
- Reordering can still drop a complete older instruction if its fragments
  arrive after a newer instruction started; SSP then relies on later state,
  as it already does for lost datagrams.
- Memory and timing bounds are unchanged.
