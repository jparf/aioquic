# Architectural Patterns

Recurring design patterns and conventions across the aioquic/pylsqpack codebase.

## 1. Event-Driven Architecture

Events are the primary communication mechanism between protocol layers and user code. Each layer defines its own event hierarchy as dataclasses inheriting from a base class.

- **QUIC events:** `quic/events.py:5` — `QuicEvent` base, subclasses like `StreamDataReceived`, `ConnectionTerminated`, `HandshakeCompleted`
- **H3 events:** `h3/events.py` — `H3Event` base, subclasses like `DataReceived`, `HeadersReceived`, `PushPromiseReceived`
- **Event queue:** `quic/connection.py:299` — `deque()` buffers events; consumed via `next_event()` at `connection.py:721`
- **Dispatch:** `asyncio/protocol.py:201` — `_process_events()` dispatches using `isinstance()` checks

Convention: Events are immutable dataclasses. Never modify an event after creation. New information = new event.

## 2. State Machine Pattern

Protocol lifecycle is modeled with explicit `Enum` states and guarded transitions.

- **QUIC states:** `quic/connection.py:191` — `QuicConnectionState` enum: `FIRSTFLIGHT → CONNECTED → CLOSING → DRAINING → TERMINATED`
- **End-state guard:** `quic/connection.py:224` — `END_STATES` frozenset used to reject operations on closed connections
- **H3 header states:** `h3/connection.py:77` — `HeadersState` enum: `INITIAL → AFTER_HEADERS → AFTER_TRAILERS`

Convention: Always check state before performing operations. Use frozensets for state groups that need membership testing.

## 3. Dataclass Value Objects

All structured data uses `@dataclass` with type-annotated fields. Mutable defaults use `field(default_factory=...)`.

- **Configuration:** `quic/configuration.py:18` — `QuicConfiguration` with 20+ typed fields
- **Events:** `quic/events.py:13` onwards — every event is a `@dataclass`
- **Packet metadata:** `quic/packet_builder.py:31` — `QuicSentPacket` with `field(default_factory=list)` for mutable collections
- **Internal state:** `quic/connection.py:183` — `QuicConnectionId`, `quic/connection.py:212` — `QuicReceiveContext`

Convention: Never use plain dicts for structured internal data. Always define a dataclass.

## 4. Strategy Pattern (Congestion Control)

Pluggable algorithms registered to a name-based factory.

- **Abstract base:** `quic/congestion/base.py:11` — `QuicCongestionControl` with abstract methods `on_packet_acked`, `on_packet_sent`, `on_packets_lost`, `on_rtt_measurement`
- **Factory function:** `quic/congestion/base.py:109` — `create_congestion_control(name, ...)` looks up `_factories` dict
- **Registration:** `quic/congestion/base.py:122` — `register_congestion_control(name, factory)` — called at module level in `reno.py` and `cubic.py`
- **Selection via config:** `quic/configuration.py:29` — `congestion_control_algorithm: str = "reno"`
- **Side-effect imports:** `quic/recovery.py:5` — `from .congestion import cubic, reno  # noqa` triggers registration

Convention: New algorithms implement the ABC and call `register_congestion_control()` at module level. The `recovery.py` import ensures all algorithms are registered.

## 5. Composition Over Inheritance

Complex objects delegate to specialized helpers rather than using deep inheritance trees.

- **QuicConnection** composes:
  - `CryptoPair` per epoch (`connection.py:292`)
  - `QuicStream` instances in a dict (`connection.py` stream management)
  - `QuicPacketRecovery` for loss detection (`recovery.py`)
  - `QuicNetworkPath` for path state (`connection.py:199`)

- **QuicStream** (`stream.py:348`) composes:
  - `QuicStreamReceiver` (`stream.py:361`)
  - `QuicStreamSender` (`stream.py:362`)

- **QuicConnectionProtocol** (`asyncio/protocol.py:12`) wraps:
  - `QuicConnection` instance (`protocol.py:23`)
  - `asyncio.StreamReader` per stream (`protocol.py:24`)

Convention: Prefer composition. The only inheritance used is for abstract interfaces (ABC for congestion control) and asyncio protocol integration.

## 6. Context Object Pattern

Groups related parameters into a single object to avoid long argument lists through the call chain.

- **QuicReceiveContext:** `quic/connection.py:212` — bundles `epoch`, `host_cid`, `network_path`, `quic_logger_frames`, `time`, `version`
- Used in `_payload_received()` and related internal methods to pass receive context through the frame processing pipeline

## 7. IntEnum Protocol Constants

All wire-protocol constants are `IntEnum` subclasses, enabling both type safety and direct use as integer values.

- **QUIC layer:** `quic/packet.py` — `QuicErrorCode` (line 27), `QuicPacketType` (line 48), `QuicProtocolVersion` (line 85), `QuicFrameType`
- **H3 layer:** `h3/connection.py` — `ErrorCode` (line 40), `FrameType` (line 64), `Setting` (line 83), `StreamType` (line 100)
- **Bidirectional mappings** for version-specific encoding: `quic/packet.py` — `PACKET_LONG_TYPE_ENCODE_VERSION_1` / `PACKET_LONG_TYPE_DECODE_VERSION_1` dicts

Convention: Never use raw integers for protocol constants. Define an `IntEnum`.

## 8. Callback-Based Decoupling

Lambda defaults and callable type aliases connect components without tight coupling.

- **Protocol callbacks:** `asyncio/protocol.py:31-37` — handlers default to no-op lambdas: `self._connection_id_issued_handler = lambda c: None`
- **Delivery handlers:** `quic/packet_builder.py:23` — `QuicDeliveryHandler = Callable[..., None]` type alias
- **Recovery probe:** `quic/recovery.py` — `send_probe: Callable[[], None]` passed to constructor

Convention: Callbacks default to no-op lambdas, not `None`. This avoids null checks at every call site.

## 9. Layered Exception Hierarchy

Each protocol layer defines its own exception hierarchy with error codes for wire transmission.

- **QUIC:** `quic/connection.py:165` — `QuicConnectionError` with `error_code`, `frame_type`, `reason_phrase`
- **QUIC streams:** `quic/stream.py:14` — `FinalSizeError`, `StreamFinishedError`
- **QUIC crypto:** `quic/crypto.py` — `CryptoError`, `KeyUnavailableError`
- **H3:** `h3/connection.py:108` — `ProtocolError` base with `error_code` attribute; 10+ subclasses (lines 108-159) for specific protocol violations
- **H3 API:** `h3/exceptions.py` — `H3Error` base with `InvalidStreamTypeError`, `NoAvailablePushIDError`

Convention: Protocol-internal errors carry error codes matching the RFC. API-facing errors are separate from wire-level errors.

## 10. Timer-Driven State (No Background Threads)

All time-dependent behavior uses cooperative timers integrated with the asyncio event loop.

- **Timer interface:** `quic/connection.py` — `get_timer()` returns next deadline; `handle_timer()` processes timeouts
- **Packet pacing:** `quic/recovery.py:34` — `QuicPacketPacer` manages send timing
- **Asyncio bridge:** `asyncio/protocol.py` — `self._loop.call_at()` schedules timer callbacks

Convention: Never use threads or `time.sleep()`. All timing goes through the timer/event-loop mechanism.

## 11. QLOG Structured Logging

Protocol events are logged in QLOG format (IETF draft) alongside standard Python logging.

- **QLOG trace:** `quic/logger.py:32` — `QuicLoggerTrace` buffers structured events as dicts in a deque
- **Context-aware logging:** `quic/connection.py:178` — `QuicConnectionAdapter` adds connection ID to all log messages
- **Standard logging:** Module-level loggers — `logging.getLogger("quic")`, `logging.getLogger("http3")`

## 12. Testing Conventions

- **Framework:** `unittest.TestCase` throughout (not pytest)
- **Async tests:** `tests/utils.py:23` — `@asynctest` decorator wraps coroutines with `asyncio.run()`
- **Test helpers:** `tests/utils.py` — Certificate generation helpers (`generate_ec_certificate`, etc.)
- **Connection helpers** in `tests/test_connection.py` — `create_standalone_client()`, `create_standalone_server()`, `client_and_server()` context manager
- **Packet loss simulation:** `tests/test_asyncio.py` — `sendto_with_loss()` monkey-patches socket

## 13. Naming Conventions

| Scope | Convention | Examples |
|-------|-----------|----------|
| Constants | `SCREAMING_SNAKE_CASE` | `K_GRANULARITY`, `STREAM_COUNT_MAX`, `END_STATES` |
| Classes | `PascalCase` with prefix | `QuicConnection`, `H3Connection`, `QuicStreamReceiver` |
| Protocol enums | `PascalCase` | `QuicErrorCode`, `FrameType`, `StreamType` |
| Public methods | `snake_case` | `next_event()`, `send_stream_data()`, `receive_datagram()` |
| Private members | `_snake_case` | `_payload_received()`, `_events`, `_cryptos` |
| Type aliases | `PascalCase` | `QuicDeliveryHandler`, `QuicTokenHandler`, `QuicStreamHandler` |

## 14. Module Organization

The codebase follows strict protocol layering:

```
quic/          ← Transport layer (QUIC protocol state machine)
  congestion/  ← Pluggable sub-component
h3/            ← Application layer (HTTP/3 built on QUIC)
h0/            ← Legacy application layer
asyncio/       ← Thin integration wrapper (not core logic)
tls.py         ← Standalone TLS 1.3 implementation
buffer.py      ← Shared low-level utilities
```

Each layer only imports from layers below it. The `asyncio/` package is a thin wrapper; all protocol logic lives in `quic/` and `h3/`. This "bring your own I/O" design makes the core testable without network access.
