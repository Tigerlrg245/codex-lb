## MODIFIED Requirements

### Requirement: Upstream websocket drops penalize affected accounts
When an upstream websocket closes while one or more streamed response requests are pending and have not reached a terminal event, the proxy MUST record a transient upstream error for the account before signaling failure for those pending requests, except when the close carries a classified process-wide network failure, upstream WebSocket liveness timeout, or an eventless `stream_incomplete` transport failure. Those exceptions MUST remain account neutral and use their classified error codes.

An eventless `stream_incomplete` failure MUST be offered to the existing bounded pre-visible replay routine. The proxy MUST replay only when that routine proves there is exactly one eligible pending request and all existing replay-safety, continuity, affinity, retry-count, and retry-circuit guards permit it. The proxy MUST NOT replay after any `response.*` event, MUST NOT replay an ambiguous continuation without a proof-gated fresh request body, and MUST NOT broaden this recovery to process-wide network or WebSocket liveness failures. If the replay gate rejects the request, the proxy MUST preserve the existing terminal `stream_incomplete` behavior and emit low-cardinality diagnostic state that explains the rejection without logging request content.

For other closes, the proxy MUST surface `stream_incomplete` to affected pending requests except when a direct Responses WebSocket request has already successfully emitted a finite integer `sequence_number`. For that sequenced direct-WebSocket case, the proxy MUST record the request outcome as `stream_incomplete` without emitting a synthetic terminal frame under the active response id, then MUST close the downstream WebSocket with code 1011.

#### Scenario: websocket closes before pending responses complete

- **GIVEN** a streamed response request is pending on an upstream websocket
- **AND** the direct downstream response has not emitted a numeric sequence, or the request uses another transport
- **WHEN** the websocket closes before a terminal response event is observed
- **AND** the close does not carry a classified process-wide network failure, upstream WebSocket liveness timeout, or safely replayable eventless `stream_incomplete`
- **THEN** the pending request fails with `stream_incomplete`
- **AND** the account receives a transient upstream failure signal for routing

#### Scenario: sequenced direct websocket closes before completion

- **GIVEN** a direct Responses WebSocket request has successfully emitted a finite integer `sequence_number`
- **WHEN** the upstream websocket closes before a terminal response event is observed
- **AND** the close does not carry a classified process-wide network failure or upstream WebSocket liveness timeout
- **THEN** the request is recorded as failed with `stream_incomplete`
- **AND** no synthetic terminal frame is emitted under the active response id
- **AND** the downstream WebSocket closes with code 1011
- **AND** the account receives a transient upstream failure signal for routing

#### Scenario: websocket liveness timeout remains account neutral

- **GIVEN** a streamed response request is pending on an upstream websocket
- **WHEN** its transport reports `upstream_websocket_liveness_timeout`
- **THEN** the pending request fails with that classified error code
- **AND** the account receives no failure-health signal
- **AND** the request is not transparently replayed

#### Scenario: safe eventless disconnect receives one bounded replay

- **GIVEN** exactly one HTTP bridge request is pending
- **AND** no `response.*` event has been observed
- **AND** the existing pre-visible replay gate proves the request safe
- **WHEN** the upstream WebSocket ends with `stream_incomplete` and no close frame
- **THEN** the proxy invokes the existing bounded replay path
- **AND** the account is not penalized
- **AND** recovery remains limited by the existing replay counter and circuit

#### Scenario: ambiguous continuation still fails closed

- **GIVEN** an HTTP bridge continuation is pending with a `previous_response_id`
- **AND** it has no proof-gated fresh replay body
- **WHEN** the upstream WebSocket ends with eventless `stream_incomplete`
- **THEN** the proxy does not resend the continuation
- **AND** it surfaces the existing terminal `stream_incomplete` failure
- **AND** the diagnostic records a replay-safety rejection without request content

#### Scenario: partial output prevents abrupt-disconnect replay

- **GIVEN** a pending request has observed any `response.*` event
- **WHEN** the upstream WebSocket ends with `stream_incomplete`
- **THEN** the proxy does not replay the request

#### Scenario: network errors remain non-replayed

- **GIVEN** the upstream failure is classified as a process-wide network failure
- **WHEN** the bridge handles the failure
- **THEN** the proxy keeps it account-neutral
- **AND** does not invoke abrupt-disconnect replay
