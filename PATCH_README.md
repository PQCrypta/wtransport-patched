# wtransport-0.6.1-patched

This is a patched fork of [wtransport](https://github.com/BiagioFesta/wtransport) v0.6.1 that fixes a race condition causing panics when scanning certain HTTP/3 servers.

## The Bug

**Symptom**: Panic with message:
```
QUIC connection is still alive on close-cast
```

**Location**: `wtransport/src/error.rs` lines 58 and 122

**Affected versions**: wtransport 0.6.1 (and likely earlier versions)

## Root Cause

The `no_connect()` and `with_no_connection()` functions call:
```rust
quic_connection.close_reason().expect("QUIC connection is still alive on close-cast")
```

`close_reason()` returns `Option<ConnectionError>`:
- `Some(reason)` - connection is closed
- `None` - connection is **still alive**

The code assumes the connection is always closed when these functions are called. However, a race condition occurs when:

1. Client connects to HTTP/3 server (e.g., Akamai-hosted sites like walgreens.com)
2. Server immediately sends SETTINGS frame on uni-directional stream
3. wtransport's internal driver thread starts processing incoming streams
4. An error triggers connection cleanup before driver completes
5. `close_reason()` returns `None` because connection is still alive
6. `.expect()` panics

## The Fix

Replace `.expect()` with graceful handling:

```rust
// BEFORE (buggy)
pub(crate) fn no_connect(quic_connection: &quinn::Connection) -> Self {
    quic_connection
        .close_reason()
        .expect("QUIC connection is still alive on close-cast")
        .into()
}

// AFTER (fixed)
pub(crate) fn no_connect(quic_connection: &quinn::Connection) -> Self {
    match quic_connection.close_reason() {
        Some(reason) => reason.into(),
        None => ConnectionError::LocallyClosed,
    }
}
```

## Reproduction

```rust
// This will panic ~50% of the time with unpatched wtransport
use wtransport::Endpoint;

let endpoint = Endpoint::client(config)?;
let connection = endpoint.connect("https://walgreens.com:443/").await?;
// Panic occurs during connection or shortly after
```

## Usage

In your `Cargo.toml`:
```toml
# Instead of:
# wtransport = "0.6"

# Use the patched version:
wtransport = { git = "https://github.com/pqcrypta/wtransport-patched", branch = "fix/close-cast-race-condition" }
```

## Upstream Status

- Issue: TBD (submit to https://github.com/BiagioFesta/wtransport/issues)
- PR: TBD

## Files Changed

- `wtransport/src/error.rs` - Fixed `no_connect()` and `with_no_connection()`
- `wtransport/Cargo.toml` - Updated version to `0.6.1-patched`, removed workspace reference

## Credits

- Original library: [BiagioFesta/wtransport](https://github.com/BiagioFesta/wtransport)
- Patch by: PQCrypta team (https://pqcrypta.com)
- Bug discovered while building HTTP/3 scanner at https://pqcrypta.com/http3-quic/
