# STPT Protocol Specification

Status: draft.

# 1	Terminology
- **Endpoint** — a running STPT instance acting as client or server.
- **Connection** — a QUIC connection secured with TLS 1.3 between a client and a server Endpoint.
- **Stream** — a QUIC stream carrying one logical channel within a Connection (control, terminal, files, other services).
- **Session** — a server-side terminal entity (PTY and its processes). Its lifetime does not depend on any Connection.
- **Attachment** — a temporary, authorized binding of a Connection to a Session.
- **Identity** — a long-term cryptographic identity of a server or a user.
- **Authentication Context** — the result of user authentication on a Connection: authenticated Identity, mapped OS account and granted permissions.
