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
- [ ] PD002  Session lifecycle: create, detach, terminate
- [ ] PD003  attach authorization, concurrent attachments
- [ ] PD004  session list
- [ ] PD005  reattach after Connection loss vs connection migration

# 5	Tier PE — File Transfer

# 6	Tier PF — Client-side File Access

# 7	Tier PG — Port Forwarding

# 8	Tier PH — Audit

# 9	Tier PI — Authentication / Identity

# 10	Tier PJ — Security Considerations

# 11	Tier PK — Interoperability / Test Vectors
