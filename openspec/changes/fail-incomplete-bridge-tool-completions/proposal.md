## Why

An upstream Responses WebSocket can finish an HTTP-bridge turn with a
`response.completed.output` snapshot that contains a supported tool-call item
without ever emitting the corresponding `response.output_item.done` event.
The public stream then advertises tool-use semantics in its terminal snapshot,
but the client never receives an executable tool-call payload. The bridge
currently settles and logs that malformed turn as success, leaving clients such
as OpenClaw with `stopReason=toolUse` and no tool payload to execute.

## What Changes

- Validate supported tool-call items in an HTTP bridge `response.completed`
  snapshot against the completed tool-call item events observed for that
  request.
- Fail closed with a downstream `response.failed` / `stream_incomplete` event
  when the terminal snapshot introduces or changes a supported tool call
  relative to the emitted completed-item manifest.
- Preserve terminal usage in failure settlement, record the request as an
  upstream failure instead of success, and use the existing reservation-first
  settlement and transient account-health path.
- Add route-level regression coverage for the malformed stream and a valid
  control stream.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `responses-api-compat`: HTTP bridge tool-use completions must have an
  executable, internally consistent streamed tool-call manifest before they
  can settle as success.

## Impact

- Code: HTTP bridge upstream terminal-event validation only.
- Tests: externally visible streaming `POST /v1/responses` bridge behavior,
  request-log outcome, usage retention, and account-health signaling.
- Existing pre-created replay/failover, downstream cancellation, and successful
  text/tool streams remain unchanged.
