# STPT Protocol Specification

Status: draft.
Specification version: draft-08 (2026-10-11). Changes: see Document history.

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

## 6.6	Port
- The default Server port is UDP 843. A Server MAY listen on another port; a Client MUST allow the port to be specified.
- The port is in the system range: on Unix-like systems an unprivileged user cannot bind it, so a local user cannot impersonate the Server on its default port. Windows has no such restriction.
- The port is not registered with IANA. If registration yields a different number, the default changes in a later specification version.

# 7	Message encoding and framing

## 7.1	Streams
- Channels use bidirectional QUIC streams only. Endpoints MUST set the transport parameter `initial_max_streams_uni` to 0.
- Every stream carries a sequence of frames (§7.2) in each direction; no stream carries unframed bytes.
- The control Channel and its stream: PB004. The first message on any other stream identifies its Channel: PB004, PC001.

## 7.2	Frame
```
frame = length payload
  length  : QUIC variable-length integer (RFC 9000 §16), the size of payload in bytes
  payload : exactly one CBOR data item, a message (§7.3)
```
- `length` MUST NOT be 0.
- A receiver MUST check `length` against its message size limit before buffering the payload. Every Endpoint MUST accept messages of at least 65535 bytes; larger limits: PB007, PB009.
- A frame MAY span several QUIC STREAM frames. The end of a stream inside a frame is a protocol violation.
- Bulk data (terminal I/O, files) travels as CBOR byte strings inside messages; a sender splits it into messages within the limit.

## 7.3	Message
Messages are encoded in CBOR (RFC 8949) and described in CDDL (RFC 8610):
```
message = [
  type: uint,              ; message type
  body: { * int => any },  ; fields with integer keys, defined per type
]
```
- Message types and fields are defined by the sections that use them. Type allocation, unknown types and extension rules: PB007.
- A receiver MUST ignore unknown keys in `body`. A field whose ignoring would affect security MUST be sent only after the capability that defines it has been negotiated (PB007).

## 7.4	CBOR rules
- A sender MUST use deterministic encoding (RFC 8949 §4.2.1): preferred serialization, definite lengths, sorted map keys.
- A receiver MUST reject a message that:
  - is not well-formed CBOR, or a payload that is not exactly one data item;
  - uses indefinite-length encoding;
  - contains duplicate map keys;
  - contains tags, floating-point numbers, or simple values other than `false`, `true` and `null`, unless the message definition allows them;
  - contains a text string that is not valid UTF-8;
  - exceeds the nesting depth limit of the receiver (PB009);
  - does not match the CDDL of its type.
- A receiver MAY reject other non-deterministic encodings.
- Rejection is a protocol violation handled per §0.2 and PB009.

# 8	Control Channel and Channel opening

## 8.1	Control stream
- The control Channel uses the first client-initiated bidirectional stream (stream ID 0). The Client opens it immediately after the handshake; its first message is HELLO (PB006).
- The control stream lives as long as the Connection. If either direction of it ends (FIN) or is reset (RESET_STREAM, STOP_SENDING), the Endpoint MUST close the Connection with an error (PB009).
- The control Channel carries connection-level messages: HELLO and negotiation (PB006, PB007), user authentication (PB008), Session operations (PD), errors and close (PB009).
- A message not permitted on the control stream, or not permitted in the current Connection state (§3.1), is a protocol violation.
- A sender SHOULD give the control stream priority over other streams.

## 8.2	Streams before authentication
Until the Connection is Authenticated only the control Channel is available (§3.1). This is enforced by QUIC stream limits:
- The Server MUST set `initial_max_streams_bidi` to 1 and MAY raise the limit with MAX_STREAMS only after the Connection is Authenticated.
- The Client MUST set `initial_max_streams_bidi` to 0 and MAY raise it only after the Connection is Authenticated, if it accepts server-initiated Channels.
- An Endpoint receiving a stream other than the control stream before the Connection is Authenticated MUST close the Connection with an error.

## 8.3	Channel opening
Every Channel other than control is opened on a new bidirectional stream. The opener's first message on it is CHANNEL_OPEN:
```
channel-open = [
  type: CHANNEL_OPEN,      ; value: PB007
  body: {
    0 => uint,             ; channel kind (terminal: PC001; others: later Tiers)
    * int => any,          ; parameters defined by the channel kind
  },
]
```
- The receiver checks the kind and the Authentication Context. To reject, it resets the stream and sends STOP_SENDING with an error code (PB009); the Connection stays open.
- After acceptance the stream carries messages of that channel kind only. Further streams of the same Channel, if any, are defined by its kind.

# 9	Server Identity and first-contact trust

## 9.1	Server Identity key
- A Server Identity is a long-term signature key pair. Endpoints MUST support Ed25519 (RFC 8032; TLS signature scheme `ed25519`). Other algorithms: later specification versions.
- The Server presents its public key in the TLS handshake as a Raw Public Key (RFC 7250): the Client MUST offer only `RawPublicKey` in `server_certificate_type`, and the Server MUST select it. X.509 certificates are not used in this version.
- The TLS handshake signature proves that the Server holds the private key of the presented public key.

## 9.2	Fingerprint
- The fingerprint of a public key is SHA-256 over its DER-encoded SubjectPublicKeyInfo.
- Text form: `SHA256:` followed by the base64 encoding of the hash without padding (RFC 4648 §4), e.g. `SHA256:47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU`.
- A Client MUST display the full fingerprint and MUST compare full keys or full fingerprints, never prefixes.

## 9.3	Trust store and TOFU
- A Client keeps a trust store mapping a Server address (the DNS name or IP address as given by the user, and the port) to a trusted public key. Entries change only by explicit user action.
- Known key: the presented key MUST equal the trust store entry; otherwise see "Changed key".
- First contact (no entry): the Client MUST either verify the key against a fingerprint supplied out of band (e.g. on the command line) or show the fingerprint and obtain explicit user confirmation. Without either, for example in non-interactive use, the Client MUST abort the Connection.
- Changed key: the Client MUST abort the Connection, MUST show the stored and the presented fingerprints, and MUST NOT offer to accept the new key in the same dialog. The user replaces the entry by a separate explicit action.
- The Client MUST NOT send any STPT message, HELLO included, before the Server key is trusted.
- Key rotation: not yet specified (DECISIONS.md §6).

# 10	HELLO and handshake sequence

## 10.1	Sequence
```
Client                                         Server
  |-- QUIC / TLS 1.3 handshake (§6) ------------>|
  |   Client verifies the Server key (§9)        |
  |-- CLIENT_HELLO (stream 0) ------------------>|
  |<--------------------------------- SERVER_HELLO|
  |   Connection Established (§3.1)              |
  |-- user authentication (PB008) -------------->|
```
- The Client sends CLIENT_HELLO as the first message on the control stream (§8.1) after the Server key is trusted, and sends nothing else until it receives SERVER_HELLO.
- The Server MUST NOT send any STPT message before it receives CLIENT_HELLO. It answers with SERVER_HELLO, or with an error followed by Connection close (PB009).
- Each Endpoint sends exactly one HELLO per Connection. A second HELLO, or any other message in its place, is a protocol violation.
- After SERVER_HELLO is sent (Server) or received (Client), the Connection is Established.
- HELLO is not signed: it travels inside TLS, which protects it against modification and downgrade.

## 10.2	Messages
Type values: PB007.
```
client-hello = [
  type: CLIENT_HELLO,
  body: {
    0 => [+ uint],                 ; supported wire versions (§5, §11.1)
    ? 1 => capabilities,           ; offered capabilities (PB007)
    ? 2 => limits,                 ; receiver limits of the Client (PB009)
    * int => any,
  },
]

server-hello = [
  type: SERVER_HELLO,
  body: {
    0 => uint,                     ; selected wire version, one of those offered
    ? 1 => capabilities,           ; selected capabilities (PB007)
    ? 2 => limits,                 ; receiver limits of the Server (PB009)
    3 => [+ uint],                 ; available user authentication methods (PB008)
    * int => any,
  },
]
```
- The Server selects the version per PB007. If no offered version is acceptable, it sends an error and closes the Connection; the Client MUST NOT retry with a different version list (§5).
- The `capabilities` and `limits` types: PB007, PB009.

## 10.3	No implementation identification
The protocol carries no implementation name or software version. Endpoints MUST NOT send such data in any message or extension field. Interoperability is determined by the wire version and capabilities only.

# 11	Negotiation and extension rules

## 11.1	Wire versions
| Range | Use |
|---|---|
| 0 | reserved (§5) |
| 0x00000001–0x7FFFFFFF | final versions |
| 0x80000000–0xFEFFFFFF | reserved |
| 0xFF000000–0xFFFFFFFF | draft versions: 0xFF000000 + N |

- N increments with every incompatible change of the wire format, not with every specification version. Current draft wire version: 0xFF000001.
- The Server selects the highest final version offered by both Endpoints; if there is none, the highest common draft version. Otherwise negotiation fails (§10.2).
- The Server MUST select a version from the CLIENT_HELLO list and MUST NOT select a GREASE value (§11.5).

## 11.2	Capabilities
```
capabilities = { * uint => any }   ; capability id => parameters
```
- The Client offers capabilities in CLIENT_HELLO; the Server selects a subset in SERVER_HELLO.
- The Server MUST NOT select a capability that was not offered. It MAY narrow the parameters within what was offered, as the capability definition allows.
- The Server ignores unknown capability ids and does not select them.
- A capability is in effect for the whole Connection only if the Server selected it.
- Capability ids: 0 is reserved; 0x00000001–0xEFFFFFFF are assigned by this specification; 0xF0000000 and above are for private and experimental use.

## 11.3	Message types
| Range | Use |
|---|---|
| 0 | reserved |
| 0x0001–0x3FFF | core protocol |
| 0x4000–0xEFFFFFFF | messages introduced by capabilities |
| 0xF0000000 and above | private and experimental use, only with a negotiated private capability |

Assigned core types:
| Type | Name | Defined in |
|---|---|---|
| 1 | CLIENT_HELLO | §10.2 |
| 2 | SERVER_HELLO | §10.2 |
| 3 | CHANNEL_OPEN | §8.3 |
| 4 | AUTH_REQUEST | §12.2 |
| 5 | AUTH_RESULT | §12.2 |

Error and close messages: PB009.

## 11.4	Extension rules
- A capability defines its id, parameters, messages, fields and channel kinds. An Endpoint MUST NOT send any of them unless the capability is in effect.
- A message of an unknown type, or of a type not in effect, is a protocol violation: the receiver MUST close the Connection (§0.2).
- Unknown keys in a message body are ignored (§7.3); unknown capability ids are ignored (§11.2).
- A capability MUST NOT lower the security of the protocol (§0.1).

## 11.5	GREASE
To keep receivers tolerant of unknown values (in the manner of RFC 8701), the following values are reserved and never assigned:
- capability ids of the form 0x?A?A (0x0A0A, 0x1A1A, …, 0xFAFA);
- wire versions of the form 0x?A?A?A?A (0x0A0A0A0A, …, 0xFAFAFAFA).

A Client MAY include such values in CLIENT_HELLO. A Server MUST ignore them and MUST NOT select them.

# 12	User authentication

## 12.1	Method
This version defines one method: public key signature with channel binding.
| Id | Method |
|---|---|
| 1 | `publickey-ed25519`: Ed25519 user key (RFC 8032) |

- A Server MUST support method 1 and lists its methods in SERVER_HELLO (§10.2).
- Password authentication is not part of the protocol.

## 12.2	Exchange
```
Client                                         Server
  |-- AUTH_REQUEST ----------------------------->|
  |<----------------------------------- AUTH_RESULT|
  |   success: Connection Authenticated (§3.1)   |
  |   failure: the Client MAY try another key    |
```
```
auth-request = [
  type: AUTH_REQUEST,
  body: {
    0 => uint,              ; method id
    1 => tstr,              ; Account name
    2 => bstr .size 32,     ; Ed25519 public key
    3 => bstr .size 64,     ; Ed25519 signature over auth-signed-data
    * int => any,
  },
]

auth-result = [
  type: AUTH_RESULT,
  body: {
    0 => bool,              ; true: success
    * int => any,
  },
]
```
- The Client sends AUTH_REQUEST only in the Established state and only one at a time: the next one after a failed AUTH_RESULT.
- There is no request to test whether a key is acceptable without a signature.

## 12.3	Channel binding
The signature covers the deterministic CBOR encoding (§7.4) of:
```
auth-signed-data = [
  "STPT user authentication",   ; context string
  bstr .size 32,                ; TLS exporter value
  uint,                         ; method id
  tstr,                         ; Account name
  bstr .size 32,                ; Ed25519 public key
]
```
- The exporter value is the TLS exporter of the Connection (RFC 8446 §7.5) with label `EXPORTER-STPT-user-auth`, empty context and length 32.
- Each Endpoint computes the exporter value itself; it is never sent. The Server rebuilds `auth-signed-data` from its own exporter value and the fields of AUTH_REQUEST.
- The exporter value is unique to the Connection and depends on its handshake, Server Identity included; a signature is therefore valid on this Connection only and cannot be relayed through another Connection.
- The Server MUST verify the signature per RFC 8032 §5.1.7.

## 12.4	Server decision
- Authentication succeeds only if the signature is valid and the public key is authorized for the named Account. How keys are authorized for Accounts is a Server matter (e.g. a per-Account list of keys).
- On success the Server creates the Authentication Context (§1), the Connection becomes Authenticated, and stream limits are raised (§8.2).
- On an Authenticated Connection a further AUTH_REQUEST is a protocol violation: the Account of a Connection never changes.
- Failure is uniform: AUTH_RESULT carries no reason, and the Server MUST NOT reveal whether the Account exists. The Server SHOULD make the response time independent of the reason.
- The Server limits failed attempts per Connection and closes the Connection when the limit is reached (PB009).

# Security Considerations
Collected from all sections; consolidated in Tier PJ.
- Ambiguous requirements cause insecure implementations; security requirements use MUST / MUST NOT (§0.1).
- Peer input is untrusted; protocol violations fail closed (§0.2).
- Version downgrade is prevented by negotiation inside TLS and by the no-fallback rule (§5).
- SNI and ALPN are visible to observers without ECH; the ALPN token reveals the protocol but not its version (§6.3, §6.4).
- The hybrid key exchange X25519MLKEM768 protects recorded traffic against future quantum attacks (§6.2).
- 0-RTT is forbidden: no replayable application data (§6.5).
- Resumption never restores user authentication (§6.2).
- On Unix-like systems the default port is privileged, so an unprivileged local user cannot occupy it (§6.6).
- Length-prefixed frames let a receiver enforce its size limit before buffering; strict CBOR rules remove parser ambiguity such as duplicate keys (§7.2, §7.4).
- Security-relevant fields are never silently ignored: they require a negotiated capability (§7.3).
- Before authentication the peer cannot open any stream but the control stream; QUIC stream limits enforce this (§8.2).
- TOFU leaves first contact open to an active attacker unless the fingerprint is verified out of band (§9.3).
- Users tend to compare fingerprint prefixes; full comparison is required (§9.2).
- Ed25519 is not secure against a quantum attacker; post-quantum Server Identity keys are an open question (DECISIONS.md §6).
- The Server stays silent until CLIENT_HELLO, and the Client sends nothing before the Server key is trusted (§9.3, §10.1).
- No implementation name or version is sent, so a scanner cannot select targets by software version (§10.3).
- Messages of unknown types close the Connection; new behaviour is possible only through negotiated capabilities (§11.4).
- User authentication is bound to the Connection by the TLS exporter, so a signature cannot be replayed or relayed (§12.3).
- No key probing and uniform failures prevent enumeration of Accounts and authorized keys (§12.2, §12.4).
- No passwords: nothing a phishing or compromised Server could capture and reuse (§12.1).

# Document history
- draft-00 (2026-10-11): conventions, terminology, concept model, state model, responsibility boundaries, versioning.
- draft-01 (2026-10-11): transport profile: QUIC, TLS 1.3, ALPN, SNI/ECH, 0-RTT (§6).
- draft-02 (2026-10-11): default port UDP 843 (§6.6).
- draft-03 (2026-10-11): message encoding and framing (§7).
- draft-04 (2026-10-11): control Channel and Channel opening (§8).
- draft-05 (2026-10-11): Server Identity and first-contact trust (§9).
- draft-06 (2026-10-11): HELLO and handshake sequence (§10).
- draft-07 (2026-10-11): negotiation and extension rules (§11); current draft wire version 0xFF000001.
- draft-08 (2026-10-11): user authentication (§12).
