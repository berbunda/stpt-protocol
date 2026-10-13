This document answers for questions:"In which order is the STPT protocol specified?"
Protocol track (P) of the roadmap (RULES.md §1.4). The realisation track (R) is in `stpt/docs/ROADMAP.md`.
An item is completed when its part of specs\PROTOCOL.md is written and its questions in specs\DECISIONS.md are decided.

# 1	Tier PA — Foundation
- [x] PA001  terminology and concept model ✅ 2026-10-11
- [x] PA002  state model and responsibility boundaries ✅ 2026-10-11
- [x] PA003  conformance language, specification and wire protocol versioning ✅ 2026-10-11

# 2	Tier PB — Transport / Connection Establishment
- [x] PB001  QUIC / TLS 1.3 profile: ALPN, SNI/ECH, 0-RTT policy ✅ 2026-10-11
- [x] PB002  default port ✅ 2026-10-11
- [x] PB003  message encoding and framing (CBOR / CDDL) ✅ 2026-10-11
- [x] PB004  control stream ✅ 2026-10-11
- [x] PB005  server identity and first-contact trust ✅ 2026-10-11
- [x] PB006  HELLO and handshake sequence ✅ 2026-10-11
- [x] PB007  version and capability negotiation, extension rules ✅ 2026-10-11
- [x] PB008  user authentication with channel binding ✅ 2026-10-11
- [x] PB009  errors, timeouts, resource limits ✅ 2026-10-11

# 3	Tier PC — Terminal Channel
- [x] PC001  terminal stream ✅ 2026-10-13
- [x] PC002  PTY parameters: terminal type, size, environment ✅ 2026-10-13
- [x] PC003  terminal I/O ✅ 2026-10-13
- [x] PC004  terminal resize ✅ 2026-10-13
- [x] PC005  signals ✅ 2026-10-13
- [x] PC006  exit status and channel close ✅ 2026-10-13

# 4	Tier PD — Session Model
- [x] PD001  Session ID and ownership ✅ 2026-10-13
- [x] PD002  Session lifecycle: create, detach, terminate ✅ 2026-10-13
- [x] PD003  attach authorization, concurrent attachments ✅ 2026-10-13
- [x] PD004  session list ✅ 2026-10-13
- [x] PD005  reattach after Connection loss vs connection migration ✅ 2026-10-13

# 5	Tier PE — File Transfer
- [ ] PE001  file transfer Channel ➕ 2026-10-13
- [ ] PE002  file operations: transfer and management, with the Account's permissions ➕ 2026-10-13
- [ ] PE003  data transfer: chunking, integrity, large files ➕ 2026-10-13
- [ ] PE004  resume of interrupted transfers ➕ 2026-10-13
- [ ] PE005  file metadata and paths across platforms ➕ 2026-10-13

# 6	Tier PF — Client-side File Access
- [ ] PF001  trust model and permission grants ➕ 2026-10-13
- [ ] PF002  requests from inside a Session to the Client ➕ 2026-10-13
- [ ] PF003  Server-opened Channel for client file access ➕ 2026-10-13
- [ ] PF004  client directory mount: explicit consent, unmount on detach or Session end ➕ 2026-10-13

# 7	Tier PG — Port Forwarding
- [ ] PG001  local forwarding; binding to Connection or Session ➕ 2026-10-13
- [ ] PG002  remote forwarding ➕ 2026-10-13
- [ ] PG003  dynamic forwarding (SOCKS) ➕ 2026-10-13
- [ ] PG004  Server forwarding policy ➕ 2026-10-13

# 8	Tier PH — Audit
- [ ] PH001  audit events ➕ 2026-10-13
- [ ] PH002  terminal I/O recording ➕ 2026-10-13
- [ ] PH003  Session command history ➕ 2026-10-13
- [ ] PH004  protection of audit records ➕ 2026-10-13

# 9	Tier PI — Authentication / Identity
- [ ] PI001  user keys on the Server: format, labels ➕ 2026-10-13
- [ ] PI002  access rights within an Account ➕ 2026-10-13
- [ ] PI003  Server key rotation ➕ 2026-10-13
- [ ] PI004  hardware keys (FIDO2) ➕ 2026-10-13
- [ ] PI005  post-quantum identity keys ➕ 2026-10-13
- [ ] PI006  certificates and CA ➕ 2026-10-13

# 10	Tier PJ — Security Considerations
- [ ] PJ001  threat model ➕ 2026-10-13
- [ ] PJ002  consolidated security considerations ➕ 2026-10-13
- [ ] PJ003  Client protection against hostile escape sequences ➕ 2026-10-13
- [ ] PJ004  denial of service analysis ➕ 2026-10-13
- [ ] PJ005  metadata leaks: keystroke timing ➕ 2026-10-13

# 11	Tier PK — Interoperability / Test Vectors
- [ ] PK001  complete CDDL schema ➕ 2026-10-13
- [ ] PK002  test vectors ➕ 2026-10-13
- [ ] PK003  conformance scenarios ➕ 2026-10-13
- [ ] PK004  criteria for version 1.0, first final wire version ➕ 2026-10-13
- [ ] PK005  IANA registration: ALPN, port ➕ 2026-10-13

# 12	Backlog — on request
Items taken up only when users ask for them.
- [ ] PG005  UDP forwarding ➕ 2026-10-13
