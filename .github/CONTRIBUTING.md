# Contributing to MinnowVPN Service

Thank you for your interest in contributing to the MinnowVPN Rust daemon! This guide covers contributions to the **core VPN implementation**.

> **Note:** For contributions to Docker deployment, see the [minnowvpn deployment repository](https://github.com/minnowvpn/minnowvpn).

## Getting Started

### Prerequisites

- Rust toolchain 1.70+ (`rustup` recommended)
- Platform-specific requirements:
  - **macOS**: Xcode Command Line Tools
  - **Linux**: `libcap-dev` for capabilities
  - **Windows**: Visual Studio Build Tools

### Development Setup

1. Fork and clone the repository:
   ```bash
   git clone https://github.com/YOUR_USERNAME/minnowvpn-service.git
   cd minnowvpn-service
   ```

2. Build the project:
   ```bash
   cargo build
   ```

3. Run tests:
   ```bash
   cargo test
   ```

4. Set up permissions for testing:
   ```bash
   # macOS
   sudo ./setuid.sh

   # Linux
   sudo setcap cap_net_admin=eip ./target/debug/minnowvpn
   ```

## Code Organization

```
src/
├── main.rs         # CLI entry point
├── client.rs       # Client mode implementation
├── server.rs       # Server mode implementation
├── crypto/         # Cryptographic primitives
├── protocol/       # WireGuard protocol layer
├── tunnel/         # TUN device abstraction
├── daemon/         # REST API daemon mode
├── config/         # Configuration parsing
└── bin/            # Verification utilities
```

## What to Contribute

### High Priority
- Bug fixes (especially protocol or platform-specific)
- Test coverage improvements
- Documentation improvements
- Performance optimizations

### Protocol Work (Careful!)
- Must maintain WireGuard compatibility
- Verify against boringtun and official WireGuard
- Run all verification binaries

### Platform Support
- Windows improvements (Wintun integration)
- Linux distribution quirks
- macOS version compatibility

## Contribution Guidelines

### Rust Code Style

- Follow standard Rust idioms
- Use `cargo fmt` before committing
- No warnings from `cargo clippy`
- Prefer explicit error handling over `.unwrap()`

### Commit Messages

Use clear, descriptive commit messages:

```
Fix replay window bitmap shift overflow

The replay window was incorrectly handling counter gaps larger
than 128 packets, causing false positive replay rejections.

- Fix bitmap shift logic for large gaps
- Add test for edge case
- Matches boringtun behavior
```

### Testing Requirements

1. **Unit tests**: Add tests for new functionality
   ```bash
   cargo test
   ```

2. **Crypto verification**: Run verification binaries
   ```bash
   cargo run --bin verify_init
   cargo run --bin verify_hmac
   cargo run --bin test_vectors
   ```

3. **Integration testing**: Test actual VPN connections
   - Client mode against official WireGuard server
   - Server mode with official WireGuard client
   - Cross-platform if possible

4. **Platform testing**: Test on your target platform(s)

### Protocol Compatibility

WireGuard compatibility is **critical**. If your change affects the protocol:

1. **Verify against boringtun**: Use `cargo run --bin compare_boringtun`
2. **Test interoperability**: Connect to/from official WireGuard
3. **Document deviations**: If intentional, explain why in the PR

### Pull Request Process

1. Create a feature branch: `git checkout -b feature/my-improvement`
2. Make your changes with clear commits
3. Ensure all tests pass: `cargo test`
4. Check for warnings: `cargo clippy`
5. Format code: `cargo fmt`
6. Push and create a Pull Request
7. Fill out the PR template completely
8. Respond to review feedback

## Module-Specific Guidelines

### crypto/
- Use established crates (no custom crypto primitives)
- Maintain HMAC RFC 2104 compatibility (not BLAKE2s keyed mode)
- Add test vectors for new functions

### protocol/
- Wire format must be byte-compatible with WireGuard
- Document any timing or state machine changes
- Update message size constants if changed

### tunnel/
- Handle all platforms (use `#[cfg(...)]` appropriately)
- Clean up routes on all code paths (including panics)
- Test route persistence across crashes

### daemon/
- Maintain REST API compatibility
- Document new endpoints in README
- Use proper HTTP status codes

## Security

- Report vulnerabilities privately (see [SECURITY.md](SECURITY.md))
- Never log sensitive data (keys, tokens)
- Use constant-time comparisons for secrets
- Review crypto code changes carefully

## Questions?

- Open a [Discussion](https://github.com/minnowvpn/minnowvpn-service/discussions) for questions
- Check existing [Issues](https://github.com/minnowvpn/minnowvpn-service/issues)

## License

By contributing, you agree that your contributions will be licensed under the Apache License 2.0.
