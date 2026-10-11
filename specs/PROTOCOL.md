# STPT Protocol Specification

Status: draft.
Specification version: draft-01 (2026-10-11). Changes: see Document history.

# 0	Conventions

## 0.1	Requirements language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals, as shown here.

Within this specification:
- Security requirements are stated with MUST or MUST NOT. SHOULD is used for them only together with the conditions under which deviation is acceptable and its consequences.
- An OPTIONAL feature MUST NOT lower the security of the protocol below that of an implementation without it.

## 0.2	Conformance
- An implementation conforms to this specification only if it satisfies every MUST and MUST NOT applicable to its role (Client, Server or both).
- All data received from a peer is untrusted and MUST be validated before use.
- Fail closed: on a protocol violation by the peer, an Endpoint MUST reject the request or close the Connection; it MUST NOT continue on a best-effort guess. Error handling details: PB007, PB009.

# 1	Terminology
- **Endpoint** — an implementation instance of the STPT protocol. Within a Connection it acts in exactly one role: Client or Server.
- **Client** — the Endpoint role that initiates a Connection.
- **Server** — the Endpoint role that accepts Connections and hosts Sessions.
- **Connection** — a secure transport connection between a Client and a Server. A Connection is not bound to a network path: an address change of an Endpoint does not create a new Connection. Mapping to QUIC and TLS 1.3: PB001.
- **Channel** — a logical service within a Connection: control, terminal, files or another service.
- **Stream** — a transport unit within a Connection carrying data of one Channel. Mapping of Channels to Streams: PB003, PB004.
- **Identity** — a long-term cryptographic identity. A **Server Identity** identifies a Server; a **User Identity** identifies a user.
- **Account** — an operating system account on the Server.
- **Authentication Context** — the result of user authentication on a Connection: authenticated User Identity, mapped Account and granted permissions.
- **Session** — a server-side execution context owned by an Account: processes and, for interactive use, a terminal (PTY). It is accessed through Channels of an attached Connection; the terminal is one such Channel. The lifetime of a Session does not depend on any Connection.
- **Attachment** — a temporary, authorized binding of a Connection to a Session. An Attachment ends no later than its Connection.
- **Retention Policy** — the Server policy determining how long a Session exists without any Attachment.

# 2	Concept model
```
Client ──── Connection ──── Server
              │  0..1 Authentication Context ── Account
              │  1..N Channel ── 1..N Stream
              │
          Attachment ──── Session ──── Account
                     (Server hosts 0..N Sessions)
```
- A Connection has exactly one Client and one Server.
- A Connection has at most one Authentication Context, established by user authentication (PB008).
- A Connection carries Channels; each Channel uses one or more Streams.
- A Server hosts zero or more Sessions; each Session is owned by exactly one Account.
- An Attachment links one Connection with one Session and requires an Authentication Context on that Connection. The number of Attachments per Connection and per Session: PD003.
- The end of a Connection ends its Channels and Attachments, but not its Sessions. A Session ends by termination or by Retention Policy (§3.2, PD002).

# 3	State model
States are abstract; their wire representation is defined in later items (PB003, PD).

## 3.1	Connection
Each Endpoint keeps its own Connection state.

| From | To | Event |
|---|---|---|
| Handshaking | Established | QUIC/TLS handshake, HELLO and negotiation succeeded |
| Established | Authenticated | user authentication succeeded; Authentication Context created |
| any | Closed | close by either Endpoint, error, idle timeout, failed handshake or authentication |

- Handshaking: QUIC/TLS handshake, HELLO, version and capability negotiation.
- Established: the channel is secure; the user is not authenticated; only the control Channel is available.
- Authenticated: an Authentication Context exists; other Channels and Attachments are available.
- Closed: final. Connection loss is a close reason, not a separate state.
- Connection migration does not change the state.

## 3.2	Session
The Server holds the authoritative Session state.

| From | To | Event |
|---|---|---|
| Starting | Running | processes started |
| Starting | Terminated | start failed |
| Running | Exited | the primary process exited |
| Running | Terminating | termination requested by an authorized Client or by Server policy |
| Terminating | Terminated | all processes ended |
| Exited | Terminated | removal by an authorized Client or by Retention Policy |

- Running: processes run; terminal I/O is available through Attachments.
- Exited: processes ended; the exit status and possibly the last output are kept for later Attachments.
- Terminated: final; resources released.
- Attached / detached is not a state: a Session is detached while it has no Attachment. A detached Session in Running or Exited is subject to Retention Policy.

## 3.3	Attachment
`Active → Ended`.
- An Attachment comes into existence Active only after successful authorization; there is no pending state.
- End reasons: detach, Connection closed, Session Terminated, permissions revoked.

## 3.4	Invariants
- An Attachment is Active only while its Connection is Authenticated and its Session is Running or Exited.
- Closing a Connection ends all its Attachments but does not change the state of any Session.
- Closed and Terminated are final; a Session ID is never reused.

# 4	Responsibility boundaries
- The Server is the authority for Session and Attachment state; the Client holds a copy.
- Each Endpoint is the authority for its own Connection state.
- The Client requests attach, detach and termination; the Server authorizes them.
- The Server alone sets and applies Retention Policy and may terminate a Session by its own policy.

# 5	Versioning
- The specification version, the wire protocol version and the application version of an implementation are independent.
- The wire protocol version is a single unsigned integer. Optional features are negotiated as capabilities, not as minor versions (PB007).
- Version 0 is reserved and MUST NOT be negotiated.
- Draft wire versions come from a range disjoint from final versions, so a draft and a final implementation never agree on a version silently. While this specification is draft, only draft versions are used. Ranges and encoding: PB007.
- Version negotiation takes place inside the TLS-protected Connection (PB006, PB007).
- An Endpoint MUST NOT retry with a lower version after a failed handshake or negotiation (downgrade protection).
- A version with a known weakness is declared deprecated; an Endpoint MUST NOT offer or accept a deprecated version.
- Each change of the specification increments the specification version and is recorded in Document history.

# 6	Transport profile
Each Connection is one QUIC connection secured by TLS 1.3.

## 6.1	QUIC
- Endpoints MUST support QUIC version 1 (RFC 9000, RFC 9001, RFC 9002). Other QUIC versions MAY be used if both Endpoints support them.
- An Endpoint MAY migrate the Connection to a new network path; a Server MAY disable active migration (`disable_active_migration`). Migration does not change the Connection state (§3.1).
- QUIC unreliable datagrams (RFC 9221) are not used.
- Idle timeout and resource limits: PB009.

## 6.2	TLS 1.3
- Cipher suites: Endpoints MUST support TLS_AES_128_GCM_SHA256 and TLS_CHACHA20_POLY1305_SHA256 and SHOULD support TLS_AES_256_GCM_SHA384. Other cipher suites MUST NOT be negotiated.
- Key exchange: Endpoints MUST support X25519 and SHOULD support the hybrid group X25519MLKEM768. If both Endpoints support X25519MLKEM768, it MUST be selected. Other groups MUST NOT be negotiated.
- Server authentication: the Server MUST authenticate in the TLS handshake with the key of its Server Identity. The Client MUST abort the handshake if verification fails. Credential form and verification: PB005.
- Client certificates are not used: the Server MUST NOT request one, and the Client MUST NOT send one. User authentication takes place after the handshake and is bound to the Connection by the TLS exporter (PB008).
- Session resumption MAY be used, only with the `psk_dhe_ke` mode. Resumption MUST NOT restore an Authentication Context: user authentication is performed on every Connection.

## 6.3	ALPN
- The ALPN token is `stpt` (0x73 0x74 0x70 0x74). The Client MUST offer it as its only protocol; the Server MUST select it or abort the handshake with `no_application_protocol`.
- The token does not carry the wire protocol version; the version is negotiated inside the Connection (§5, PB007). The token changes only if the HELLO encoding changes incompatibly.

## 6.4	SNI and ECH
- If the Server is addressed by a DNS name, the Client SHOULD send it in the `server_name` extension. An IP address MUST NOT be sent as SNI (RFC 6066).
- SNI and ALPN travel in cleartext unless Encrypted Client Hello (ECH) is used. The Client SHOULD use ECH if an ECH configuration for the Server is available (e.g. a DNS HTTPS record, RFC 9460).
- ECH rejection is handled as defined by the ECH specification. A Client configured to use ECH for a Server MUST NOT retry the Connection without ECH.
- Server-side ECH is OPTIONAL.

## 6.5	0-RTT
- The Client MUST NOT send 0-RTT data. The Server MUST NOT accept 0-RTT data and MUST NOT issue session tickets that allow early data.
- Reason: 0-RTT data is replayable and lacks forward secrecy; saving one round trip does not justify it. A later version MAY allow it through a capability (PB007).

# Security Considerations
Collected from all sections; consolidated in Tier PJ.
- Ambiguous requirements cause insecure implementations; security requirements use MUST / MUST NOT (§0.1).
- Peer input is untrusted; protocol violations fail closed (§0.2).
- Version downgrade is prevented by negotiation inside TLS and by the no-fallback rule (§5).
- SNI and ALPN are visible to observers without ECH; the ALPN token reveals the protocol but not its version (§6.3, §6.4).
- The hybrid key exchange X25519MLKEM768 protects recorded traffic against future quantum attacks (§6.2).
- 0-RTT is forbidden: no replayable application data (§6.5).
- Resumption never restores user authentication (§6.2).

# Document history
- draft-00 (2026-10-11): conventions, terminology, concept model, state model, responsibility boundaries, versioning.
- draft-01 (2026-10-11): transport profile: QUIC, TLS 1.3, ALPN, SNI/ECH, 0-RTT (§6).
