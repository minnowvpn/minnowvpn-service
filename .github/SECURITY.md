# Security Policy

## Supported Versions

MinnowVPN Service is currently in **alpha** development. Security updates are provided for the latest release only.

| Version | Supported          |
|---------|--------------------|
| 1.x.x   | :white_check_mark: |
| < 1.0   | :x:                |

## Reporting a Vulnerability

**Please do NOT report security vulnerabilities through public GitHub issues.**

### How to Report

Use GitHub's private vulnerability reporting feature:

1. Go to the [Security tab](https://github.com/minnowvpn/minnowvpn-service/security)
2. Click "Report a vulnerability"
3. Fill out the form with details

### What to Include

- Description of the vulnerability
- Steps to reproduce the issue
- Potential impact (what could an attacker do?)
- Affected code/module if known
- Suggested fix (if you have one)

### What to Expect

- **Initial Response**: Within 48 hours
- **Status Update**: Within 7 days
- **Fix Timeline**: Critical issues within 7 days, others within 30 days

### Scope

Security issues we're especially interested in:

#### Cryptographic Issues
- Weaknesses in Noise IKpsk2 implementation
- HMAC/KDF construction errors
- Key handling vulnerabilities
- Nonce reuse or predictability
- Replay attack vectors not covered by the sliding window

#### Protocol Issues
- Handshake bypass or manipulation
- Session hijacking possibilities
- DoS amplification attacks
- Cookie mechanism bypasses

#### Implementation Issues
- Memory safety issues (use-after-free, buffer overflows)
- Command injection in route management
- Privilege escalation
- Token/authentication bypass in daemon mode
- Secrets exposure in logs or state files

### Out of Scope

- Denial of Service requiring significant resources
- Issues requiring physical access
- Social engineering
- Issues in third-party dependencies (please report upstream)

## Security Design

MinnowVPN Service implements several security measures:

### Cryptography
- Uses well-audited crates (x25519-dalek, chacha20poly1305, blake2)
- HMAC follows RFC 2104 (verified against boringtun)
- Automatic session rekey every 120 seconds
- 128-packet replay protection window

### Authentication
- Daemon mode uses 32-byte random tokens
- Token files have restricted permissions (0640 on Unix)
- Bearer token required for all API requests

### State Management
- Route state files allow deterministic cleanup
- Connection state encrypted at rest (in progress)
- Graceful shutdown removes all routes

## Verification

You can verify the cryptographic implementation using the included test binaries:

```bash
cargo run --bin verify_init    # Verify Noise constants match boringtun
cargo run --bin verify_hmac    # HMAC test vectors
cargo run --bin test_vectors   # WireGuard test vectors
```

Thank you for helping keep MinnowVPN secure!
