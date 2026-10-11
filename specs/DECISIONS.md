# STPT Design Decisions

Status: working record of design decisions and open questions. Not a normative wire protocol specification.
Origin: translated from the Russian summary "STPT_Decisions_v0.1" (2026-10-10) and extended with later decisions.
Everything here is open to discussion unless fixed in docs/RULES.md.

Marks: ✅ decided, ❓ open. Each mark carries a date.

# 1	Purpose
STPT is a standalone secure tool for remote terminal access and related operations, intended as an alternative to SSH.
The goal is not wire compatibility with SSH, but keeping its successful ideas while improving the session model and adding capabilities.
Do not create differences for the sake of differences.

- ❓ Final expansion of the name: "Secure Terminal Processor Tools" or "Secure Terminal Processing Tools".

# 2	Repositories and licensing
- ✅ 2026-10-10 Two independent repositories: `stpt` (Rust application, GPLv3) and `stpt-protocol` (protocol description, CC0 1.0).
- `stpt-protocol` contains documentation only. Formal message schemas, examples and test vectors are allowed later.
- The specification must be sufficient to build an independent implementation without reading the Rust sources.
- The protocol specification (`stpt-protocol/specs`) and the description of the implementation (`stpt/docs/ARCHITECTURE.md`) must not substitute for each other.

# 3	Platforms
- Application language: Rust.
- Primary platforms: Windows 11 x86-64 and Debian-based Linux x86-64.
- One source project; each platform build is a single binary acting as client and server, mode selected via CLI. Builds are platform-specific.
- Interactive terminal: PTY on Linux; ConPTY for a future Windows server mode.

# 4	Architectural model
- Endpoint, Client, Server, Connection, Channel, Stream, Identity, Account, Authentication Context, Session, Attachment and Retention Policy are distinct concepts (see PROTOCOL.md §1, §2).
- ✅ 2026-10-11 A Session is a server-side execution context owned by an Account, not only a terminal; the terminal is one of its Channels.
- ✅ 2026-10-11 Persistence is not a kind of Session: every Session is independent of Connections, and its life without an Attachment is set by the server Retention Policy. A new Connection may attach to it after authorization.
- ✅ 2026-10-11 Channel (logical service: control, terminal, files, other) and Stream (transport unit) are separate terms; the mapping is defined in PB003/PB004.
- ✅ 2026-10-11 A Connection is not bound to a network path: QUIC connection migration keeps the Connection; after Connection loss a new Connection is required.
- SSH already multiplexes channels. The QUIC advantage is, among others, independent loss recovery per stream.

# 5	Transport and connection establishment
- ✅ 2026-10-10 QUIC over UDP with TLS 1.3. Candidate Rust library: `quinn` (with `rustls`).
- Order: QUIC/TLS handshake and server identity verification → STPT HELLO → version and capability negotiation → user authentication → Session create/attach.
- No STPT banner or any application data is sent before the cryptographic channel is established.
- This does not make the protocol invisible: IP addresses, traffic metadata and QUIC itself remain observable. QUIC Initial packets are only obfuscated (their keys are derivable by anyone).
- ✅ 2026-10-10 ALPN and SNI travel in cleartext in the ClientHello. Where this exposes too much, Encrypted Client Hello (ECH) is to be used. Server-side ECH is restricted by gate G1.
- ✅ 2026-10-10 No application data in 0-RTT (replayable). At most HELLO; never terminal input, attach, file or other state-changing operations.
- QUIC connection migration (address change) and Session recovery after full connection loss are different mechanisms; both are in scope.
- ❓ Default port. Options:
  - a dedicated UDP port (note: 22000 is already used by Syncthing, including QUIC);
  - sharing UDP 443 with HTTP/3 and demultiplexing by ALPN (requires a front end that routes QUIC Initials; interacts with ECH, since ALPN is then inside the encrypted inner ClientHello).

# 6	Authentication and security
- No custom cryptographic algorithms; only proven TLS and cryptographic implementations.
- Server identity: TOFU model with SSH known_hosts-like UX; connection refused on unexpected identity change.
- ✅ 2026-10-10 User authentication happens at the STPT application layer after the TLS handshake. The client proves possession of its key by signing a value obtained from the TLS exporter of the current Connection (channel binding), so the proof cannot be reused on another Connection.
- Short-lived certificates, CA, hybrid post-quantum key exchange: promising, not required for the first MVP.
- Authorization of Session attach and of client file system operations must be defined explicitly.
- The server never gets access to client files merely because a Connection exists.
- ❓ Security model of mounting a client directory on the server (reverse mount) requires a separate analysis.

# 7	Features
Core:
- secure connection to a remote terminal;
- full interactive PTY session: resize, control signals, terminal applications;
- persistent sessions: create, list, detach, reattach, terminate.

Later:
- native file transfer over STPT;
- requesting client files from a remote terminal session, with permissions only;
- mounting a client directory on the server (Linux FUSE; Windows needs a separate solution);
- session audit and terminal I/O recording;
- port forwarding;
- additional identity, deployment and hardening models.

Keystroke recording is not a reliable log of executed shell commands; secret handling and audit boundaries need separate design.

# 8	CLI
- ✅ 2026-10-10 Commands are explicit: every operation has its own subcommand; no implicit forms such as `stpt user@host`.
- Preliminary shape (syntax not fixed):
  ```
  stpt server --listen <addr>:<port>
  stpt connect user@host[:port] [--new | --attach <session-id>]
  stpt sessions user@host
  stpt exec user@host "command"
  stpt fs pull|push|mount ...
  stpt keygen
  ```

# 9	Versioning and terminology
- Application version, wire protocol version, Tier and Gate are different entities.
- ✅ 2026-10-11 The roadmap is split into two tracks: protocol (P, docs/ROADMAP.md) and realisation (R, stpt/docs/ROADMAP.md). R items depend on P items explicitly; the protocol is specified ahead of the implementation.
- Tiers, items and gates: see docs/RULES.md §1.4, docs/ROADMAP.md, stpt/docs/ROADMAP.md, docs/GATES.md.

# 10	MVP on a local VM
First verifiable result:
- STPT server on a local Debian VM, STPT client on Windows 11;
- secure QUIC/TLS connection, server identity verification, minimal user authentication;
- bash through a PTY: interactive I/O, resize, Ctrl+C, nano/htop work;
- clean session termination.

Next experiment:
- open a Session, start a long-running process, drop the Connection;
- verify the process and PTY survive;
- open a new Connection and attach to the same Session.

# 11	Open questions before protocol work
- ✅ 2026-10-10 Message encoding: CBOR (RFC 8949) with CDDL schemas (RFC 8610).
- ❓ Frame format, message types, protocol version and extension rules.
- ❓ Server identity, first-contact trust, key rotation.
- ❓ User authentication methods, mapping to OS accounts, access rights.
- ❓ Session ID, ownership, lifetimes, attach/detach, concurrent access.
- ❓ Resource limits, timeouts, error handling, DoS protection.
- ❓ How a command inside the remote shell reaches the client without breaking trust boundaries.
- ❓ Audit format and security, including password and secret input.
- ❓ Interoperability of independent implementations and test vectors.

# 12	Working principle
First define the state model and responsibility boundaries, then the wire format, only then the implementation.
Start with a narrow MVP verifiable on a local VM.
