# Phase 2 Implementation Plan

## Overview

Two new files: `research/controllable_h3.py` and `research/controllable_client.py`, plus
`research/tests/test_controllable.py` for integration tests against the local example server.

---

## File 1: `research/controllable_h3.py` — ControllableH3Connection

Subclass of `H3Connection` that adds manual QPACK encoder stream injection.

### Key Design Decisions

**Problem**: `H3Connection.__init__` calls `_init_connection()` immediately (line 421),
which creates the encoder stream. We need the connection fully initialized before we can
inject manual instructions. This is fine — we just add methods that use the already-created
`_local_encoder_stream_id`.

**Problem**: When in manual mode, the server sends QPACK decoder stream acknowledgments
for our manually-inserted entries. The `pylsqpack.Encoder` doesn't know about them and
will raise `DecoderStreamError`. We must intercept this.

**Problem**: `_encoder.apply_settings()` is called at line 713 when SETTINGS arrive. This
automatically sends a Set Dynamic Table Capacity instruction. In manual mode, we want to
control this ourselves, but we should let the automatic one go through and just track what
capacity was set — our ManualQpackEncoder needs to know the server's max table capacity.

### Class: ControllableH3Connection(H3Connection)

```python
class ControllableH3Connection(H3Connection):
    def __init__(self, quic, *, manual_encoder=False, **kwargs):
        self._manual_mode = manual_encoder
        self._manual_decoder_log: list[bytes] = []  # log decoder acks in manual mode
        super().__init__(quic, **kwargs)

    def send_encoder_instruction(self, instruction_bytes: bytes) -> None:
        """Write raw bytes to the QPACK encoder stream."""
        self._quic.send_stream_data(
            self._local_encoder_stream_id, instruction_bytes
        )

    @property
    def peer_max_table_capacity(self) -> int:
        """Return server's QPACK_MAX_TABLE_CAPACITY from received SETTINGS, or 0."""
        if self._received_settings is None:
            return 0
        return self._received_settings.get(Setting.QPACK_MAX_TABLE_CAPACITY, 0)
```

**Override `_receive_stream_data_uni`**: Only in manual mode, intercept the QPACK decoder
stream data (stream_type == QPACK_DECODER). Instead of calling `self._encoder.feed_decoder()`,
log the raw bytes. For all other stream types, delegate to `super()`.

The challenge is that `_receive_stream_data_uni` is a monolithic method that handles all
uni stream types in one function. We can't easily override just the decoder stream part.

**Approach**: Override the full method BUT only intercept when:
  - `stream.stream_type == StreamType.QPACK_DECODER` AND `self._manual_mode is True`

For that case, consume the bytes and log them. For all other cases, call `super()`.

Actually, looking more carefully at the code, `_receive_stream_data_uni` identifies the
stream type on first call and stores it in `stream.stream_type`. Once identified, on
subsequent calls it dispatches by type. The cleanest approach:

**Revised approach**: Don't override `_receive_stream_data_uni` at all. Instead, monkey-patch
or wrap `self._encoder.feed_decoder` to be a no-op in manual mode. This is much simpler
and less fragile.

We can do this by replacing `self._encoder` with a wrapper, or more simply by overriding
`_receive_stream_data_uni` only for the decoder stream branch. But actually the cleanest:

**Final approach**: Create a `_PermissiveEncoder` wrapper that delegates to the real encoder
but swallows `feed_decoder()` errors in manual mode, logging the raw bytes instead. This
avoids duplicating the entire `_receive_stream_data_uni` method.

Actually, the simplest: just replace `_encoder.feed_decoder` with a lambda/method that logs:

```python
if self._manual_mode:
    original_feed_decoder = self._encoder.feed_decoder
    def _intercept_feed_decoder(data):
        self._manual_decoder_log.append(data)
    self._encoder.feed_decoder = _intercept_feed_decoder
```

This is clean, minimal, and doesn't duplicate any H3Connection code.

---

## File 2: `research/controllable_client.py` — ControllableHttpClient + factory

### Class: ControllableHttpClient(QuicConnectionProtocol)

Follows the `HttpClient` pattern from `examples/http3_client.py`:

```python
class ControllableHttpClient(QuicConnectionProtocol):
    def __init__(self, *args, manual_encoder=False, **kwargs):
        super().__init__(*args, **kwargs)
        self._manual_encoder = manual_encoder
        self._http = ControllableH3Connection(
            self._quic,
            manual_encoder=manual_encoder,
        )
        self._request_events = {}
        self._request_waiter = {}

    def quic_event_received(self, event):
        for http_event in self._http.handle_event(event):
            self.http_event_received(http_event)

    def http_event_received(self, event):
        # Same pattern as HttpClient: collect events, resolve waiters
        ...

    async def get(self, url, headers=None):
        ...

    async def _request(self, method, url, headers, content=None):
        # Same as HttpClient._request but simpler (no websocket/push)
        ...

    def send_encoder_instruction(self, instruction_bytes):
        """Delegate to ControllableH3Connection."""
        self._http.send_encoder_instruction(instruction_bytes)
        self.transmit()

    @property
    def peer_max_table_capacity(self):
        return self._http.peer_max_table_capacity
```

### Factory: create_controllable_client

Async context manager following the `connect()` pattern from `asyncio/client.py`:

```python
@asynccontextmanager
async def create_controllable_client(host, port, *, configuration=None,
                                      manual_encoder=False, **kwargs):
    # Same as aioquic.asyncio.client.connect() but uses ControllableHttpClient
    # as create_protocol, passing manual_encoder through
    ...
```

The tricky part is passing `manual_encoder` to the protocol constructor. The `connect()`
function uses `create_protocol` as a callable. We can use `functools.partial` or a lambda.

---

## File 3: `research/tests/test_controllable.py`

Tests that run against the example server (`examples/http3_server.py`).

### Test structure:
1. **Start server in a subprocess** using `examples/http3_server.py` with the test SSL certs
2. **Create controllable client**, connect to localhost
3. **Verify**: basic GET works
4. **Verify**: `send_encoder_instruction()` with valid bytes accepted by server
5. **Verify**: `peer_max_table_capacity` returns server's setting
6. **Verify**: GET after manual instruction still works (connection not broken)

### Server management:
- Start server in setUp/setUpClass using subprocess
- Use `tests/ssl_cert.pem` and `tests/ssl_key.pem` for TLS
- Kill server in tearDown/tearDownClass

---

## Implementation Order

1. `research/controllable_h3.py` — ControllableH3Connection
2. `research/controllable_client.py` — ControllableHttpClient + create_controllable_client
3. `research/tests/test_controllable.py` — integration tests
4. Run tests, iterate on failures
