## 1. Specification and diagnosis

- [x] 1.1 Correlate the malformed bridge completion with the client-visible
      tool-use/no-payload outcome and define the supported tool-call manifest
      invariant.
- [x] 1.2 Add the `responses-api-compat` delta before implementation.

## 2. Implementation

- [x] 2.1 Track and compare the supported tool-call manifest from
      `response.output_item.done` events with the terminal
      `response.completed.output` snapshot.
- [x] 2.2 Rewrite inconsistent completions to `response.failed` with
      `stream_incomplete`, preserving usage for settlement and request logs.
- [x] 2.3 Preserve existing reservation-before-health ordering, transient
      account failure handling, cancellation, and replay/failover eligibility.

## 3. Regression coverage

- [x] 3.1 Add an externally routed streaming `/v1/responses` regression proving
      a terminal-only tool call is not emitted or recorded as success.
- [x] 3.2 Assert failure usage/account-health behavior and a valid tool-call
      stream control.

## 4. Validation

- [x] 4.1 Run focused HTTP bridge tests and relevant broader proxy tests.
- [x] 4.2 Run Ruff, type checks, and strict OpenSpec validation.
