# MinnowVPN Service

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Alpha-orange.svg)]()
[![Under Development](https://img.shields.io/badge/Under%20Development-⚠️-yellow.svg)]()

> ⚠️ **Alpha Software**: This project is under active development and not yet ready for production use. APIs and features may change without notice. Use at your own risk.

> **Note:** This is a reference repository. Bug reports are welcome, but pull requests will not be accepted.

**Small. Fast. Lean.** A modern WireGuard VPN daemon written in Rust.

## Overview

Like its namesake, MinnowVPN is deliberately small and nimble. No bloat, no unnecessary features—just a fast, focused VPN that does one thing well.

MinnowVPN Service is the core daemon for the MinnowVPN project, implementing the complete WireGuard protocol (Noise IKpsk2 handshake pattern). It operates in three modes:

- **Client Mode** - Connect to a WireGuard VPN server
- **Server Mode** - Accept incoming VPN client connections
- **Daemon Mode** - Run as a background service with REST API for GUI control

The implementation is verified against Cloudflare's [boringtun](https://github.com/cloudflare/boringtun) to ensure protocol compatibility.

## Why MinnowVPN?

| | |
|---|---|
| 🐟 **Small** | ~11,000 lines of focused Rust code. Single binary, no runtime dependencies. The entire daemon fits in a few megabytes. |
| ⚡ **Fast** | Native Rust performance with sub-second handshakes. Efficient ChaCha20-Poly1305 encryption. Minimal memory footprint. |
| 🎯 **Lean** | Does one thing well: WireGuard VPN. No unnecessary abstractions, no feature creep. Clean, auditable codebase. |

## Features

- **WireGuard Protocol** - Full Noise IKpsk2 implementation with automatic rekey
- **Cross-Platform** - Runs on macOS, Linux, and Windows
- **Daemon Mode** - HTTP REST API with Bearer token authentication
- **Auto-Reconnect** - Persistent state survives daemon restarts and reboots
- **Multi-Peer** - Server mode supports multiple simultaneous clients
- **DoS Protection** - Cookie mechanism for rate limiting
- **Replay Protection** - 128-packet sliding window

## Building

### Prerequisites

- Rust toolchain (1.70+)
- Platform-specific:
  - **macOS**: Xcode Command Line Tools
  - **Linux**: `libcap` for capabilities (`sudo apt install libcap-dev`)
  - **Windows**: Visual Studio Build Tools

### Build Commands

```bash
# Debug build
cargo build

# Release build (optimized)
cargo build --release

# Run tests
cargo test
```

### Platform Setup

```bash
# macOS: Set up setuid permissions (after each build)
sudo ./setuid.sh

# Linux: Grant network capability (alternative to running as root)
sudo setcap cap_net_admin=eip ./target/release/minnowvpn
```

## CLI Usage

```
minnowvpn [OPTIONS]

Options:
  -c, --config <FILE>     Path to WireGuard configuration file
  -v, --verbose           Enable debug-level logging
      --client            Force client mode
      --server            Force server mode
      --daemon            Run as daemon with REST API
      --http-port <PORT>  HTTP port for daemon mode (default: 51820)
      --token-path <PATH> Custom path for auth token file
  -h, --help              Print help
  -V, --version           Print version
```

### Examples

```bash
# Client mode (connect to VPN server)
sudo ./minnowvpn -c client.conf

# Server mode (accept connections)
sudo ./minnowvpn -c server.conf --server

# Client mode with verbose logging
sudo ./minnowvpn -c client.conf -v

# Daemon mode (for GUI control)
sudo ./minnowvpn --daemon

# Daemon mode on custom port
sudo ./minnowvpn --daemon --http-port 51821
```

### Mode Auto-Detection

If neither `--client` nor `--server` is specified, the mode is auto-detected:

- **Server mode**: Config has `ListenPort` AND no peer has `Endpoint`
- **Client mode**: At least one peer has an `Endpoint`
- **Ambiguous**: Requires explicit `--server` or `--client` flag

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Configuration error |
| 2 | Insufficient privileges |
| 3 | Network error |
| 4 | Protocol error |
| 5 | Cryptography error |
| 255 | Other error |

## Daemon REST API

The daemon runs an HTTP server on `127.0.0.1:{port}` with Bearer token authentication.

### Authentication

On startup, the daemon generates a 32-byte random token and writes it to a protected file:

| Platform | Token Path |
|----------|------------|
| Unix | `/var/run/minnowvpn/auth-token` |
| Windows | `C:\ProgramData\MinnowVPN\auth-token` |

All requests require the header:
```
Authorization: Bearer <token>
```

### Client Mode Endpoints

#### POST /api/v1/connect

Start VPN connection.

```json
// Request
{"config": "<wireguard-config-content>"}

// Response (200 OK)
{"connected": true}
```

#### POST /api/v1/disconnect

Stop VPN connection.

```json
// Response (200 OK)
{"disconnected": true}
```

#### GET /api/v1/status

Get connection status.

```json
// Response (200 OK)
{
  "state": "connected",
  "vpn_ip": "10.200.0.9",
  "server_endpoint": "vpn.example.com:51820",
  "bytes_sent": 1024,
  "bytes_received": 2048
}
```

#### PUT /api/v1/config

Update configuration dynamically (validates before reconnecting).

```json
// Request
{"config": "<new-wireguard-config>"}

// Response (200 OK)
{
  "updated": true,
  "vpn_ip": "10.200.0.9",
  "server_endpoint": "vpn.example.com:51820"
}
```

### Server Mode Endpoints

#### POST /api/v1/server/start

Start VPN server.

```json
// Request
{"config": "<wireguard-server-config>"}

// Response (200 OK)
{"started": true}
```

#### POST /api/v1/server/stop

Stop VPN server.

```json
// Response (200 OK)
{"stopped": true}
```

#### GET /api/v1/server/peers

List all configured peers.

```json
// Response (200 OK)
{
  "peers": [
    {
      "public_key": "<base64>",
      "allowed_ips": ["10.0.0.2/32"],
      "has_session": true,
      "bytes_sent": 5120,
      "bytes_received": 10240
    }
  ]
}
```

#### POST /api/v1/server/peers

Add a peer dynamically.

```json
// Request
{
  "public_key": "<base64>",
  "allowed_ips": ["10.0.0.2/32"],
  "preshared_key": "<base64>"  // optional
}

// Response (200 OK)
{"added": true, "public_key": "<base64>"}
```

#### DELETE /api/v1/server/peers/:pubkey

Remove a peer.

```json
// Response (200 OK)
{"removed": true, "public_key": "<base64>"}
```

### Server-Sent Events

#### GET /api/v1/events

Real-time event stream (SSE).

**Client Mode Events:**
- `status_changed` - Connection state changes
- `config_updated` - Config applied successfully
- `config_update_failed` - Config rollback occurred
- `auto_connect_retry` - Reconnection attempt

**Server Mode Events:**
- `server_status_changed` - Server state changes
- `peer_connected` - Peer completed handshake
- `peer_disconnected` - Peer session ended
- `peer_added` - Peer added via API
- `peer_removed` - Peer removed via API

## Configuration Format

Standard WireGuard `.conf` format:

### Client Configuration

```ini
[Interface]
PrivateKey = <base64-private-key>
Address = 10.200.0.9/24
DNS = 8.8.8.8

[Peer]
PublicKey = <base64-server-public-key>
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

### Server Configuration

```ini
[Interface]
PrivateKey = <base64-private-key>
Address = 10.100.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <base64-client-public-key>
AllowedIPs = 10.100.0.2/32

[Peer]
PublicKey = <base64-another-client-key>
AllowedIPs = 10.100.0.3/32
```

### Configuration Fields

**[Interface]**
| Field | Required | Description |
|-------|----------|-------------|
| `PrivateKey` | Yes | Base64-encoded X25519 private key |
| `Address` | Yes | Interface IP address(es) in CIDR notation |
| `ListenPort` | Server only | UDP port for incoming connections |
| `DNS` | No | DNS server(s), space or comma-separated |
| `MTU` | No | Interface MTU (default: 1420) |

**[Peer]**
| Field | Required | Description |
|-------|----------|-------------|
| `PublicKey` | Yes | Base64-encoded X25519 public key |
| `Endpoint` | Client only | Server address:port |
| `AllowedIPs` | Yes | CIDR ranges to route through this peer |
| `PresharedKey` | No | Optional additional symmetric key |
| `PersistentKeepalive` | No | Keepalive interval in seconds |

## Architecture

```
src/
├── main.rs              # CLI entry point, mode detection, signal handling
├── lib.rs               # Library exports
├── client.rs            # Client mode event loop (537 lines)
├── server.rs            # Server mode event loop (837 lines)
├── error.rs             # Error type hierarchy
├── config/              # WireGuard .conf parser
│   ├── mod.rs
│   └── parser.rs
├── crypto/              # Cryptographic primitives
│   ├── mod.rs
│   ├── blake2s.rs       # BLAKE2s hash, HMAC, KDF
│   ├── aead.rs          # ChaCha20-Poly1305 encryption
│   ├── x25519.rs        # X25519 Diffie-Hellman
│   └── noise.rs         # Noise IKpsk2 state machine
├── protocol/            # WireGuard protocol layer
│   ├── mod.rs
│   ├── messages.rs      # Wire format (Handshake, Transport, Cookie)
│   ├── handshake.rs     # Initiator & Responder handshake
│   ├── transport.rs     # Packet encryption/decryption
│   ├── session.rs       # Session state, PeerManager, rekey
│   └── cookie.rs        # DoS protection (MAC2)
├── tunnel/              # Cross-platform TUN device
│   └── mod.rs           # TunDevice, RouteManager
├── daemon/              # REST API daemon mode
│   ├── mod.rs           # DaemonService, HTTP server
│   ├── ipc.rs           # Request/response DTOs
│   ├── auth.rs          # Bearer token authentication
│   ├── routes.rs        # HTTP route handlers
│   └── persistence.rs   # State persistence for auto-reconnect
└── bin/                 # Debug & verification utilities
```

### Code Statistics

| Module | Lines | Purpose |
|--------|-------|---------|
| daemon/ | 3,777 | REST API and state management |
| protocol/ | 1,856 | WireGuard protocol |
| tunnel/ | 1,099 | TUN device abstraction |
| server.rs | 837 | Server event loop |
| crypto/ | 666 | Cryptographic primitives |
| client.rs | 537 | Client event loop |
| config/ | 416 | Configuration parsing |
| **Total** | ~11,300 | Complete implementation |

## Protocol Implementation

### Noise IKpsk2 Pattern

The handshake follows the Noise protocol framework:

```
Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s

Initiator                          Responder
---------                          ---------
-> e, es, s, ss
                                   <- e, ee, se, psk
```

### Cryptographic Primitives

| Algorithm | Usage |
|-----------|-------|
| X25519 | Diffie-Hellman key exchange |
| ChaCha20-Poly1305 | AEAD encryption (handshake & transport) |
| XChaCha20-Poly1305 | Cookie encryption (24-byte nonce) |
| BLAKE2s | Hashing and MAC |
| HMAC-BLAKE2s | Key derivation (RFC 2104) |

### Message Formats

| Type | Name | Size | Description |
|------|------|------|-------------|
| 1 | Handshake Initiation | 148 bytes | Client initiates connection |
| 2 | Handshake Response | 92 bytes | Server accepts connection |
| 3 | Cookie Reply | 64 bytes | DoS protection token |
| 4 | Transport Data | 16 + payload | Encrypted IP packets |

### Session Management

- **Rekey**: Automatic after 120 seconds
- **Replay Protection**: 128-packet sliding window
- **Keepalive**: Configurable via `PersistentKeepalive`
- **Counter Limit**: 2^64 - 8192 packets per session

## Boringtun Compatibility

The implementation is verified against Cloudflare's [boringtun](https://github.com/cloudflare/boringtun), a widely-used Rust WireGuard implementation.

### Key Compatibility Points

- **HMAC Construction**: Uses RFC 2104 `SimpleHmac<Blake2s256>` (not BLAKE2s keyed mode) - critical for interoperability
- **Initial Constants**: Chain key and hash values verified to match boringtun exactly
- **DH Order**: Diffie-Hellman operations in handshake follow identical sequence

### Verification Binaries

```bash
# Compare crypto output with boringtun
cargo run --bin compare_boringtun

# Verify initial Noise constants
cargo run --bin verify_init

# Run test vectors from wireguard-go and boringtun
cargo run --bin test_vectors
```

| Binary | Purpose |
|--------|---------|
| `compare_boringtun` | Compare crypto output with boringtun |
| `verify_init` | Verify initial chain key/hash constants |
| `verify_hmac` | HMAC-BLAKE2s test vectors |
| `verify_noise` | Noise state machine verification |
| `verify_aead` | ChaCha20-Poly1305 encryption tests |
| `test_vectors` | wireguard-go and boringtun test vectors |
| `deterministic_test` | Reproducible crypto verification |

## Platform-Specific Details

### TUN Device

| Platform | Interface | Notes |
|----------|-----------|-------|
| macOS | `utun{N}` | Requires root/sudo |
| Linux | `tun0` | Requires `CAP_NET_ADMIN` or root |
| Windows | Wintun | Requires Administrator |

### State File Locations

| Platform | Auth Token | Connection State | Routes |
|----------|------------|------------------|--------|
| Unix | `/var/run/minnowvpn/auth-token` | `/var/lib/minnowvpn/connection-state.json` | `/var/run/minnowvpn_routes.json` |
| Windows | `C:\ProgramData\MinnowVPN\auth-token` | `C:\ProgramData\MinnowVPN\connection-state.json` | `C:\ProgramData\MinnowVPN\routes.json` |

### Route Management

Routes are tracked in a state file for crash recovery:
- On startup: Checks for orphaned routes from previous crashes
- On shutdown: Cleans up all added routes
- Endpoint bypass: VPN endpoint routed through default gateway to prevent loops

## Dependencies

### Cryptography
- `x25519-dalek` - X25519 Diffie-Hellman
- `chacha20poly1305` - ChaCha20-Poly1305 AEAD
- `blake2` - BLAKE2s hashing
- `hmac` - HMAC construction

### Networking
- `tokio` - Async runtime
- `tun-rs` - Cross-platform TUN device
- `axum` - HTTP framework (daemon mode)

### Utilities
- `clap` - CLI argument parsing
- `serde` / `serde_json` - Serialization
- `tracing` - Structured logging

## Related Projects

- [MinnowVPN](https://github.com/minnowvpn/minnowvpn) - Docker deployment with web management console
- [boringtun](https://github.com/cloudflare/boringtun) - Cloudflare's WireGuard implementation (compatibility reference)

## License

Licensed under the Apache License, Version 2.0. See [LICENSE](LICENSE) for details.
