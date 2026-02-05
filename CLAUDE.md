# QUIC Research Workspace

Two complementary Python libraries implementing QUIC (RFC 9000/9369) and HTTP/3 (RFC 9114):

- **aioquic** (v1.3.0) — Full QUIC protocol and HTTP/3 implementation with asyncio integration
- **pylsqpack** (v0.3.23) — Python wrapper around the ls-qpack QPACK header compression C library

Author: Jeremy Lainé (aiortc project). License: BSD-3-Clause.

## Tech Stack

- **Language:** Python 3.10+ with extensive type hints
- **C extensions:** `_buffer.c`, `_crypto.c` (aioquic); `binding.c` wrapping vendored `ls-qpack` (pylsqpack)
- **Dependencies:** cryptography, pyopenssl, pylsqpack, certifi, service-identity
- **Build:** setuptools with ABI3 stable-API wheels, cibuildwheel for cross-platform
- **Lint/Format:** ruff (E, F, W, I rules)
- **Type checking:** mypy (strict mode) — see `aioquic/pyproject.toml:50`
- **Tests:** unittest (not pytest)
- **Docs:** Sphinx with autodoc

## Project Structure

```
99/
├── aioquic/                        # Main QUIC/HTTP3 library
│   ├── src/aioquic/
│   │   ├── quic/                   # Core QUIC protocol
│   │   │   ├── connection.py       # QuicConnection state machine (main entry)
│   │   │   ├── configuration.py    # QuicConfiguration dataclass
│   │   │   ├── packet.py           # Packet parsing, enums, wire format
│   │   │   ├── packet_builder.py   # Packet construction helper
│   │   │   ├── stream.py           # QuicStream (receiver + sender)
│   │   │   ├── recovery.py         # Loss detection, RTT, pacing
│   │   │   ├── congestion/         # Pluggable congestion control (reno, cubic)
│   │   │   ├── crypto.py           # CryptoPair encryption/decryption
│   │   │   ├── events.py           # QUIC event dataclasses
│   │   │   ├── logger.py           # QLOG-format structured logging
│   │   │   └── rangeset.py         # Acknowledged ranges data structure
│   │   ├── h3/                     # HTTP/3 layer
│   │   │   ├── connection.py       # H3Connection (QPACK, frames, streams)
│   │   │   ├── events.py           # H3 event dataclasses
│   │   │   └── exceptions.py       # H3-specific errors
│   │   ├── h0/                     # HTTP/0.9 (minimal/legacy)
│   │   ├── asyncio/                # Async integration layer
│   │   │   ├── protocol.py         # QuicConnectionProtocol (DatagramProtocol)
│   │   │   ├── client.py           # connect() async context manager
│   │   │   └── server.py           # serve() async server
│   │   ├── tls.py                  # Minimal TLS 1.3 implementation
│   │   ├── _buffer.c / _crypto.c   # Performance-critical C extensions
│   │   └── buffer.py               # Buffer utilities
│   ├── tests/                      # Comprehensive unittest suite
│   ├── examples/                   # HTTP/3 client/server, DNS-over-QUIC, etc.
│   └── docs/                       # Sphinx documentation source
├── pylsqpack/                      # QPACK wrapper library
│   ├── src/pylsqpack/
│   │   ├── binding.c               # C extension wrapping ls-qpack
│   │   └── __init__.pyi            # Type stubs
│   ├── tests/                      # Encoder/decoder/roundtrip tests
│   └── vendor/ls-qpack/            # Vendored LiteSpeed QPACK C library
└── quic-research.code-workspace    # VS Code workspace config
```

## Build & Test Commands

All commands run from the `aioquic/` directory unless noted.

### Install
```bash
pip install .              # from source
pip install .[dev]         # with dev dependencies (coverage)
```

### Test
```bash
python -m unittest discover -v          # run all tests
coverage run -m unittest discover -v    # with coverage
coverage xml                            # generate coverage report
```

### Lint & Type Check
```bash
ruff check .                     # lint
ruff format --check --diff .     # format check
mypy examples src tests          # type check (strict)
check-manifest                   # verify MANIFEST.in
```

### Docs
```bash
make -C docs html SPHINXOPTS=-W  # build docs (warnings as errors)
```

### pylsqpack (from `pylsqpack/` directory)
```bash
pip install .
python -m unittest discover -v
ruff check .
```

## Key Entry Points

| What | Where |
|------|-------|
| QUIC connection state machine | `aioquic/src/aioquic/quic/connection.py:233` — `QuicConnection` |
| HTTP/3 protocol layer | `aioquic/src/aioquic/h3/connection.py` — `H3Connection` |
| Configuration | `aioquic/src/aioquic/quic/configuration.py:19` — `QuicConfiguration` |
| Async client | `aioquic/src/aioquic/asyncio/client.py` — `connect()` |
| Async server | `aioquic/src/aioquic/asyncio/server.py` — `serve()` |
| Congestion control registry | `aioquic/src/aioquic/quic/congestion/base.py:109` — `create_congestion_control()` |
| QUIC events | `aioquic/src/aioquic/quic/events.py:5` — `QuicEvent` base |
| H3 events | `aioquic/src/aioquic/h3/events.py` — `H3Event` base |
| Test helpers | `aioquic/tests/utils.py:23` — `asynctest()` decorator |

## CI

GitHub Actions (`.github/workflows/tests.yml`): lint, mypy, codespell, tests on Python 3.10-3.14 across Ubuntu/macOS/Windows, wheel building, PyPI publish on tags.

## Additional Documentation

Check these files for deeper context when working in specific areas:

- [Architectural Patterns](.claude/docs/architectural_patterns.md) — Event-driven design, state machines, strategy pattern, composition, dataclass conventions, and other recurring patterns across the codebase
