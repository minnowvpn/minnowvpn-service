# MinnowVPN Service

[![License](https://img.shields.io/badge/License-Proprietary-red.svg)]()
[![Status](https://img.shields.io/badge/Status-Alpha-orange.svg)]()
[![Under Development](https://img.shields.io/badge/Under%20Development-⚠️-yellow.svg)]()

> ⚠️ **Alpha Software**: This project is under active development and not yet ready for production use. APIs and features may change without notice. Use at your own risk.

**A WireGuard-compatible VPN daemon written in Rust.**

This is the core VPN daemon for MinnowVPN. It implements the WireGuard protocol (Noise IKpsk2 handshake) and provides both client and server modes with a REST API for control.

## Features

- WireGuard-compatible protocol implementation
- Client and server modes
- Daemon mode with REST API for GUI/service control
- Cross-platform: macOS, Linux, Windows
- Automatic session rekey and keepalive

## Building

```bash
cargo build --release
```

## Usage

```bash
# Client mode
sudo ./target/release/minnowvpn -c client.conf

# Server mode
sudo ./target/release/minnowvpn -c server.conf --server

# Daemon mode (for GUI control)
sudo ./target/release/minnowvpn --daemon
```

## Related

- [MinnowVPN](https://github.com/minnowvpn/minnowvpn) - Docker deployment with web console
