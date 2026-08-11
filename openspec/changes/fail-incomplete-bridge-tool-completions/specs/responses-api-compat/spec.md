# responses-api-compat Delta

## ADDED Requirements

### Requirement: HTTP bridge tool completions match emitted tool-call items

Before the HTTP Responses bridge accepts an upstream `response.completed`
event as successful, every supported durable tool-call item in the terminal
`response.output` snapshot (`function_call`, `custom_tool_call`, or
`apply_patch_call`) MUST match the complete supported tool-call manifest emitted
for the same request through `response.output_item.done` events. The manifest
comparison MUST use both call identity and item type and MUST reject introduced,
duplicated, or type-changed supported tool calls. Completed-item tool calls that
are absent from a non-empty terminal snapshot remain executable because their
payload was already emitted and are not invalidated by this one-way check.

An inconsistent completion MUST NOT be forwarded as `response.completed` and
MUST NOT be settled, logged, or reported to account health as success. The
bridge MUST instead emit a retryable `response.failed` event with
`error.code = "stream_incomplete"`, preserve any valid upstream terminal usage
for reservation settlement and request logging, settle the reservation before
writing the transient account-health failure, and retire the affected upstream
socket. A malformed completion received after response ownership or model
output became visible MUST NOT be transparently replayed; pre-created requests
that remain eligible for the existing bounded replay/failover paths are
unchanged.

Terminal responses whose `output` is missing or empty MAY continue to use the
existing public-stream backfill from completed item events. Unsupported durable
tool item types remain governed by their existing handling.

#### Scenario: Terminal-only function call fails closed

- **GIVEN** an HTTP bridge request has emitted `response.created`
- **AND** no completed function-call item was emitted for that request
- **WHEN** `response.completed.output` contains a `function_call`
- **THEN** the downstream stream emits `response.failed` with
  `error.code = "stream_incomplete"` instead of `response.completed`
- **AND** no synthetic executable tool-call item is fabricated
- **AND** the request log records an upstream error rather than success

#### Scenario: Malformed tool completion preserves settlement inputs

- **GIVEN** the malformed terminal response carries valid usage
- **WHEN** the bridge finalizes the rewritten failure
- **THEN** input, cached-input, output, and reasoning token usage remain
  available to reservation settlement and request logging
- **AND** reservation settlement completes before the transient account-health
  failure is written

#### Scenario: Streamed function call completes normally

- **GIVEN** a function call with the same call id and item type was emitted in a
  valid `response.output_item.done` event for the request
- **WHEN** `response.completed.output` contains that function call
- **THEN** the bridge forwards the successful completion unchanged
- **AND** normal success accounting and health handling apply
