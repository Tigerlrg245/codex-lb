## Why

The HTTP Responses bridge treats an abrupt upstream WebSocket disconnect as an
account-neutral transport failure and skips the existing pre-visible replay
routine entirely. As a result, even a request that the replay-safety gate can
prove safe fails immediately with `stream_incomplete` when the socket disappears
before `response.created`. This differs from clean-close and silent-timeout
handling and makes transient network hiccups terminate otherwise recoverable
agent turns.

## What Changes

- Route an eventless `stream_incomplete` upstream disconnect through the existing
  bounded pre-visible replay routine.
- Preserve the existing replay-safety gates, hard-affinity behavior, circuit
  limits, and no-replay behavior after any response event.
- Keep process-wide network and WebSocket liveness failures account-neutral and
  non-replayed.
- Add regression tests and low-cardinality diagnostics for replayed and rejected
  eventless disconnects.

## Impact

Eligible self-contained requests can survive one abrupt eventless WebSocket
hiccup. Ambiguous continuations without a proof-gated fresh body still fail
closed, and recovery remains bounded by the existing replay counter and retry
circuit.
