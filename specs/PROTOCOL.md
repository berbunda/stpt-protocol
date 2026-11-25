# STPT Protocol Specification

Status: draft.
Specification version: draft-61 (2026-11-25). Changes: see Document history.

# 0	Conventions

## 0.1	Requirements language
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals, as shown here.

Within this specification:
- Security requirements are stated with MUST or MUST NOT. SHOULD is used for them only together with the conditions under which deviation is acceptable and its consequences.
- An OPTIONAL feature MUST NOT lower the security of the protocol below that of an implementation without it.

## 0.2	Conformance
- An implementation conforms to this specification only if it satisfies every MUST and MUST NOT applicable to its role (Client, Server or both).
- All data received from a peer is untrusted and MUST be validated before use.
- Fail closed: on a protocol violation by the peer, an Endpoint MUST reject the request or close the Connection; it MUST NOT continue on a best-effort guess. Error handling details: §11, §13.

# 1	Terminology
- **Endpoint** — an implementation instance of the STPT protocol. Within a Connection it acts in exactly one role: Client or Server.
- **Client** — the Endpoint role that initiates a Connection.
- **Server** — the Endpoint role that accepts Connections and hosts Sessions.
- **Connection** — a secure transport connection between a Client and a Server. A Connection is not bound to a network path: an address change of an Endpoint does not create a new Connection. Mapping to QUIC and TLS 1.3: §6.
- **Channel** — a logical service within a Connection: control, terminal, files or another service.
- **Stream** — a transport unit within a Connection carrying data of one Channel. Mapping of Channels to Streams: §7, §8.
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
- A Connection has at most one Authentication Context, established by user authentication (§12).
- A Connection carries Channels; each Channel uses one or more Streams.
- A Server hosts zero or more Sessions; each Session is owned by exactly one Account.
- An Attachment links one Connection with one Session and requires an Authentication Context on that Connection. A Connection has at most one Attachment per Session; a Session may have several (§15.9).
- The end of a Connection ends its Channels and Attachments, but not its Sessions. A Session ends by exit, termination or Retention Policy (§3.2, §15.6–§15.8).

# 3	State model
States are abstract; their wire representation is defined in §7 and later sections (PC, PD).

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
| Exited | Terminated | exit status delivered to an Attachment, removal by an authorized Client or by Retention Policy |

- Running: processes run; terminal I/O is available through Attachments.
- Exited: processes ended; the exit status and possibly the last output are kept for later Attachments.
- Terminated: final; resources released.
- Attached / detached is not a state: a Session is detached while it has no Attachment. A detached Session in Running or Exited is subject to Retention Policy.

## 3.3	Attachment
`Active → Ended`.
- An Attachment comes into existence Active only after successful authorization; there is no pending state.
- End reasons: detach (by its Client, by a detach request or by a takeover), Connection closed, stalled stream, Session Terminated, permissions revoked.

## 3.4	Invariants
- An Attachment is Active only while its Connection is Authenticated and its Session is Running or Exited.
- Closing a Connection ends all its Attachments but does not change the state of any Session.
- Closed and Terminated are final; a Session ID is never reused (§15.1).

# 4	Responsibility boundaries
- The Server is the authority for Session and Attachment state; the Client holds a copy.
- Each Endpoint is the authority for its own Connection state.
- The Client requests attach, detach and termination; the Server authorizes them.
- The Server alone sets and applies Retention Policy and may terminate a Session by its own policy.

# 5	Versioning
- The specification version, the wire protocol version and the application version of an implementation are independent.
- The wire protocol version is a single unsigned integer. Optional features are negotiated as capabilities, not as minor versions (§11).
- Version 0 is reserved and MUST NOT be negotiated.
- Draft wire versions come from a range disjoint from final versions, so a draft and a final implementation never agree on a version silently. While this specification is draft, only draft versions are used. Ranges and encoding: §11.
- Version negotiation takes place inside the TLS-protected Connection (§10, §11).
- An Endpoint MUST NOT retry with a lower version after a failed handshake or negotiation (downgrade protection).
- A version with a known weakness is declared deprecated; an Endpoint MUST NOT offer or accept a deprecated version.
- Each change of the specification increments the specification version and is recorded in Document history.

# 6	Transport profile
Each Connection is one QUIC connection secured by TLS 1.3.

## 6.1	QUIC
- Endpoints MUST support QUIC version 1 (RFC 9000, RFC 9001, RFC 9002). Other QUIC versions MAY be used if both Endpoints support them.
- An Endpoint MAY migrate the Connection to a new network path. A Server MUST NOT disable active migration (`disable_active_migration`), so a Client keeps its Connection across a network change (§15.12). Migration does not change the Connection state (§3.1).
- QUIC unreliable datagrams (RFC 9221) are not used.
- Idle timeout and resource limits: §13.

## 6.2	TLS 1.3
- Cipher suites: Endpoints MUST support TLS_AES_128_GCM_SHA256 and TLS_CHACHA20_POLY1305_SHA256 and SHOULD support TLS_AES_256_GCM_SHA384. Other cipher suites MUST NOT be negotiated.
- Key exchange: Endpoints MUST support X25519 and SHOULD support the hybrid group X25519MLKEM768. If both Endpoints support X25519MLKEM768, it MUST be selected. Other groups MUST NOT be negotiated.
  - A Client that supports X25519MLKEM768 MUST list it before X25519 in `supported_groups` and SHOULD send a key share for it.
  - A Server MUST NOT select X25519 if the Client offers X25519MLKEM768. If the Client lists X25519 first, the Server selects X25519MLKEM768 (with HelloRetryRequest if needed) or aborts the handshake.
- Server authentication: the Server MUST authenticate in the TLS handshake with the key of its Server Identity. The Client MUST abort the handshake if verification fails. Credential form and verification: §9.
- Client certificates are not used: the Server MUST NOT request one, and the Client MUST NOT send one. User authentication takes place after the handshake and is bound to the Connection by the TLS exporter (§12).
- Session resumption MAY be used, only with the `psk_dhe_ke` mode. Resumption MUST NOT restore an Authentication Context: user authentication is performed on every Connection.

## 6.3	ALPN
- The ALPN token is `stpt` (0x73 0x74 0x70 0x74). The Client MUST offer it as its only protocol; the Server MUST select it or abort the handshake with `no_application_protocol`.
- The token does not carry the wire protocol version; the version is negotiated inside the Connection (§5, §11). The token changes only if the HELLO encoding changes incompatibly.

## 6.4	SNI and ECH
- If the Server is addressed by a DNS name, the Client SHOULD send it in the `server_name` extension. An IP address MUST NOT be sent as SNI (RFC 6066).
- SNI and ALPN travel in cleartext unless Encrypted Client Hello (ECH) is used. The Client SHOULD use ECH if an ECH configuration for the Server is available (e.g. a DNS HTTPS record, RFC 9460).
- ECH rejection is handled as defined by the ECH specification. A Client configured to use ECH for a Server MUST NOT retry the Connection without ECH.
- Server-side ECH is OPTIONAL.

## 6.5	0-RTT
- The Client MUST NOT send 0-RTT data. The Server MUST NOT accept 0-RTT data and MUST NOT issue session tickets that allow early data.
- Reason: 0-RTT data is replayable and lacks forward secrecy; saving one round trip does not justify it. A later version MAY allow it through a capability (§11).

## 6.6	Port
- The default Server port is UDP 843. A Server MAY listen on another port; a Client MUST allow the port to be specified.
- The port is in the system range: on Unix-like systems an unprivileged user cannot bind it, so a local user cannot impersonate the Server on its default port. Windows has no such restriction.
- The port is not registered with IANA. If registration yields a different number, the default changes in a later specification version.

# 7	Message encoding and framing

## 7.1	Streams
- Channels use bidirectional QUIC streams only. Endpoints MUST set the transport parameter `initial_max_streams_uni` to 0.
- Every stream carries a sequence of frames (§7.2) in each direction; no stream carries unframed bytes.
- The control Channel and its stream: §8. The first message on any other stream identifies its Channel: §8, §14.

## 7.2	Frame
```
frame = length payload
  length  : QUIC variable-length integer (RFC 9000 §16), the size of payload in bytes
  payload : exactly one CBOR data item, a message (§7.3)
```
- `length` MUST NOT be 0.
- A receiver MUST check `length` against its message size limit before buffering the payload. Every Endpoint MUST accept messages of at least 65535 bytes; larger limits: §11, §13.
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
- Message types and fields are defined by the sections that use them. Type allocation, unknown types and extension rules: §11.
- A receiver MUST ignore unknown keys in `body`. A field whose ignoring would affect security MUST be sent only after the capability that defines it has been negotiated (§11).

## 7.4	CBOR rules
- A sender MUST use deterministic encoding (RFC 8949 §4.2.1): preferred serialization, definite lengths, sorted map keys.
- A receiver MUST reject a message that:
  - is not well-formed CBOR, or a payload that is not exactly one data item;
  - uses indefinite-length encoding;
  - contains duplicate map keys;
  - contains tags, floating-point numbers, or simple values other than `false`, `true` and `null`, unless the message definition allows them;
  - contains a text string that is not valid UTF-8;
  - exceeds the nesting depth limit of the receiver (§13);
  - does not match the CDDL of its type.
- A receiver MAY reject other non-deterministic encodings.
- Rejection is a protocol violation handled per §0.2 and §13.

# 8	Control Channel and Channel opening

## 8.1	Control stream
- The control Channel uses the first client-initiated bidirectional stream (stream ID 0). The Client opens it immediately after the handshake; its first message is HELLO (§10).
- The control stream lives as long as the Connection. If either direction of it ends (FIN) or is reset (RESET_STREAM, STOP_SENDING), the Endpoint MUST close the Connection with an error (§13).
- The control Channel carries connection-level messages: HELLO and negotiation (§10, §11), user authentication (§12), Session requests (§8.4, §15).
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
  type: CHANNEL_OPEN,      ; value 3 (§11.3)
  body: {
    0 => uint,             ; channel kind (§14.1)
    * int => any,          ; parameters defined by the channel kind
  },
]
```
- The receiver checks the kind and the Authentication Context. To reject, it resets the stream and sends STOP_SENDING with CHANNEL_REJECTED (§13.2); the Connection stays open.
- After acceptance the stream carries messages of that channel kind only. Further streams of the same Channel, if any, are defined by its kind.
- A stream whose first message is not CHANNEL_OPEN, or that ends (FIN) before CHANNEL_OPEN, is a protocol violation. A stream the opener resets before CHANNEL_OPEN is abandoned and ignored.

## 8.4	Requests
Operations that are not Channels (e.g. Session termination, §15.6) are requests on the control Channel:
```
request = [
  type: uint,              ; request type, e.g. SESSION_TERMINATE
  body: {
    0 => request-id,
    * int => any,          ; parameters defined by the request
  },
]

request-id = uint

request-result = [
  type: REQUEST_RESULT,    ; value 29 (§11.3)
  body: {
    0 => request-id,
    1 => uint,             ; error code (§13.2); 0 (NO_ERROR): success
    * int => any,          ; result data defined by the request
  },
]
```
- In this version only the Client sends requests, and only on an Authenticated Connection.
- The Client chooses the request ID; it MUST be unique among its requests still without a result. A duplicate is a protocol violation.
- The Server answers every request with exactly one REQUEST_RESULT. Results may come in any order, so a long request does not hold up others.
- The Server MAY limit outstanding requests per Connection (RECOMMENDED 16) and answers a request over the limit with LIMIT_EXCEEDED.
- A failed request is reported only in its result; the Connection stays open. A REQUEST_RESULT with an unknown request ID is a protocol violation.

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
  |-- user authentication (§12) -------------->|
```
- The Client sends CLIENT_HELLO as the first message on the control stream (§8.1) after the Server key is trusted, and sends nothing else until it receives SERVER_HELLO.
- The Server MUST NOT send any STPT message before it receives CLIENT_HELLO. It answers with SERVER_HELLO or closes the Connection with an error code (§13.1).
- Each Endpoint sends exactly one HELLO per Connection. A second HELLO, or any other message in its place, is a protocol violation.
- After SERVER_HELLO is sent (Server) or received (Client), the Connection is Established.
- HELLO is not signed: it travels inside TLS, which protects it against modification and downgrade.

## 10.2	Messages
Type values: §11.
```
client-hello = [
  type: CLIENT_HELLO,
  body: {
    0 => [+ uint],                 ; supported wire versions (§5, §11.1)
    ? 1 => capabilities,           ; offered capabilities (§11)
    ? 2 => limits,                 ; receiver limits of the Client (§13)
    * int => any,
  },
]

server-hello = [
  type: SERVER_HELLO,
  body: {
    0 => uint,                     ; selected wire version, one of those offered
    ? 1 => capabilities,           ; selected capabilities (§11)
    ? 2 => limits,                 ; receiver limits of the Server (§13)
    3 => [+ uint],                 ; available user authentication methods (§12)
    * int => any,
  },
]
```
- The Server selects the version per §11. If no offered version is acceptable, it closes the Connection with VERSION_NEGOTIATION_FAILED (§13.2); the Client MUST NOT retry with a different version list (§5).
- The `capabilities` and `limits` types: §11, §13.

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

Assigned capabilities: 1 — `file-transfer` (§16.2), 2 — `client-file-access` (§17.5), 3 — `client-directory-mount` (§17.8), 4 — `tcp-forwarding` (§18.2).

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
| 6 | TERMINAL_ATTACHED | §14.3 |
| 7 | TERMINAL_DATA | §14.7 |
| 8 | TERMINAL_STDERR | §14.7 |
| 24 | TERMINAL_EOF | §14.7 |
| 25 | TERMINAL_RESIZE | §14.8 |
| 26 | TERMINAL_SIGNAL | §14.9 |
| 27 | TERMINAL_EXIT | §14.10 |
| 28 | SESSION_TERMINATE | §15.6 |
| 29 | REQUEST_RESULT | §8.4 |
| 30 | SESSION_DETACH | §15.9 |
| 31 | SESSION_LIST | §15.10 |

Assigned capability message types (capability 1): 0x4000 — FILES_READY (§16.2); 0x4001 — FILE_REQUEST, 0x4002 — FILE_RESULT, 0x4003 — FILE_ENTRIES, 0x4004 — FILE_ACCEPTED (§16.5); 0x4005 — FILE_DATA, 0x4006 — FILE_END, 0x4007 — FILE_RECEIVED (§16.10).

Capability 2 assigns 0x4010–0x4017 to the kind-3 exchange (§17.5–§17.7). Capability 3 assigns 0x4018–0x401A to directory mounts (§17.8–§17.10). Capability 4 assigns 0x4020–0x4022 to TCP forwarding (§18.2). There are no Connection error or close messages: errors use QUIC mechanisms (§13.1).

## 11.4	Extension rules
- A capability defines its id, parameters, messages, fields and channel kinds. An Endpoint MUST NOT send any of them unless the capability is in effect.
- A message of an unknown type, or of a type not in effect, is a protocol violation: the receiver MUST close the Connection (§0.2).
- Unknown keys in a message body are ignored (§7.3); unknown capability ids are ignored (§11.2).
- A capability MUST NOT lower the security of the protocol (§0.1).

## 11.5	GREASE
To keep receivers tolerant of unknown values (in the manner of RFC 8701), the following values are reserved and never assigned:
- capability ids of the form 0x?A?A, where both `?` are the same hexadecimal digit: 0x0A0A, 0x1A1A, …, 0xFAFA (16 values);
- wire versions of the form 0x?A?A?A?A, where all `?` are the same hexadecimal digit: 0x0A0A0A0A, 0x1A1A1A1A, …, 0xFAFAFAFA (16 values).

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
- Failure is uniform: AUTH_RESULT carries no reason, and the Server MUST NOT reveal whether the Account exists.
- The Server MUST perform every check of AUTH_REQUEST (signature verification with the key from the request, Account lookup, key authorization) whichever of them fails, and decide once after all of them. The signature is verified even if the Account does not exist. The work, and so the response time, does not depend on the reason of a failure.
- The Server limits failed attempts per Connection and closes the Connection with AUTH_FAILED when the limit is reached (§13.4).

# 13	Errors, timeouts and resource limits

## 13.1	Error signalling
Errors use QUIC mechanisms; there is no error message:
- Connection error: CONNECTION_CLOSE (application variant) with an error code (§13.2). A regular close uses NO_ERROR.
- Stream error: RESET_STREAM and STOP_SENDING with an error code; the Connection stays open.
- Errors of individual control requests (e.g. Session termination) are reported in REQUEST_RESULT (§8.4); file operation outcomes use FILE_RESULT (§16.5).
- Reason phrase of CONNECTION_CLOSE:
  - A close initiated by the STPT implementation (every application close, and a transport close it chooses itself) MUST have an empty reason phrase before the Connection is Authenticated. Afterwards the reason phrase MAY contain a short diagnostic text and MUST NOT contain secrets.
  - Diagnostic text that the QUIC or TLS implementation adds on its own to a transport close (e.g. after a failed TLS handshake) is outside this rule. An implementation SHOULD suppress it before authentication where its QUIC or TLS implementation allows.
  - The reason phrase is for diagnostics only: a receiver MUST NOT derive behaviour from it.
- The Server SHOULD record the exact reason of every rejection (Connection, authentication, Channel, request) in its local log. The peer receives only the error code; the administrator diagnoses from the log.

## 13.2	Error codes
| Code | Name | Scope | Meaning |
|---|---|---|---|
| 0 | NO_ERROR | Connection, stream | regular close |
| 1 | PROTOCOL_VIOLATION | Connection | peer violated this specification (§0.2) |
| 2 | INTERNAL_ERROR | Connection, stream | local failure |
| 3 | VERSION_NEGOTIATION_FAILED | Connection | no common wire version (§11.1) |
| 4 | AUTH_FAILED | Connection | authentication attempts exhausted (§12.4) |
| 5 | TIMEOUT | Connection, stream | a protocol deadline expired (§13.3, §16.2, §16.13) |
| 6 | LIMIT_EXCEEDED | Connection, stream, request | a protocol or configured resource limit exceeded (§13.4, §16) |
| 7 | CHANNEL_REJECTED | stream | CHANNEL_OPEN rejected (§8.3) |
| 8 | SERVER_SHUTDOWN | Connection | the Server is shutting down |
| 9 | SESSION_NOT_FOUND | stream, request | no Session with this ID for the Account (§15.2) |
| 10 | SESSION_NAME_IN_USE | stream | the Session name is taken (§15.3) |
| 11 | ATTACHMENT_NOT_FOUND | request | no Attachment with this ID in the Session (§15.9) |

Unknown error codes are treated as INTERNAL_ERROR. Codes 0xF0000000 and above are for private and experimental use.

## 13.3	Timeouts
| Deadline | Default | On expiry |
|---|---|---|
| CLIENT_HELLO received by the Server after the handshake | 10 s | Server closes with TIMEOUT |
| SERVER_HELLO received by the Client after it sent CLIENT_HELLO | 10 s | Client closes with TIMEOUT |
| AUTH_RESULT received by the Client after it sent AUTH_REQUEST | 10 s | Client closes with TIMEOUT |
| Connection Authenticated after it becomes Established | 60 s | Server closes with TIMEOUT |
| QUIC idle timeout (`max_idle_timeout`) | 60 s | Connection closed silently (RFC 9000 §10.1) |
| progress of a stalled terminal stream | 10 s | Server ends the Attachment with TIMEOUT (§15.9) |
| FILES_READY received after CHANNEL_OPEN | 10 s | Client aborts only the file Channel with TIMEOUT (§16.2) |
| progress of a file operation | 60 s | Endpoint aborts only the file Channel with TIMEOUT (§16.13) |
| Client file access: opening / consent / start / transfer progress | 10 / 60 / 10 / 60 s | Abort only the affected Channel (§17.7) |
| Forwarding opening / SOCKS handshake / blocked progress | 10 / 10 / 60 s | Abort the affected rule or flow (§18.7); healthy idle has no progress timer |

- Defaults are RECOMMENDED; a Server MAY configure other values.
- A Client SHOULD send QUIC PING frames when the Connection has been idle for 15 s, so an idle terminal keeps its Connection. Session survival does not depend on this (§3.4).

## 13.4	Resource limits
```
limits = {
  ? 0 => uint,    ; maximum message size in bytes accepted by the sender of HELLO
  * int => any,
}
```
- Maximum message size: default and minimum 65535 bytes (§7.2). An Endpoint MUST NOT send a message larger than the limit announced by the peer.
- CBOR nesting depth: every Endpoint MUST accept a depth of 16; messages defined by this specification never exceed it.
- Account name: at most 256 bytes.
- Terminal parameters and environment: §14.6.
- Outstanding requests per Connection: §8.4. Sessions per Account: §15.8. Attachments per Session: §15.9.
- File Channels, open resources, staging and transfer sizes: §16.2, §16.9–§16.10, §16.13 and §16.16.
- Server-opened client-file-access Channels and pending prompts: §17.5–§17.7.
- Forwarding rules, flows, pending sockets, DNS and buffered bytes: §18.2, §18.7.
- Failed authentication attempts per Connection: default 6; a Server MAY use a lower value.
- A violated limit is answered with LIMIT_EXCEEDED.

## 13.5	Denial of service
- A Server SHOULD limit the number of Connections that are not yet Authenticated, in total and per source address.
- A Server SHOULD use QUIC Retry (RFC 9000 §8.1.2) to validate client addresses under load.
- Before authentication a Server keeps only the state needed for the handshake; Session resources are allocated only after authentication.

# 14	Terminal Channel

## 14.1	Channel kinds
| Kind | Channel | Defined in |
|---|---|---|
| 0 | reserved | |
| 1 | terminal | §14 |
| 2 | files (capability 1) | §16 |
| 3 | client-file-access (capability 2) | §17 |
| 4 | forward-rule (capability 4) | §18.4 |
| 5 | forward-flow (capability 4) | §18.5 |

Control uses stream 0 (§8.1). Future services may assign additional kinds.

## 14.2	Purpose and modes
- A terminal Channel is the Attachment of a Connection to a Session (§3.3): opening it attaches, closing it detaches.
- A Session has at most one terminal; several terminals are several Sessions.
- Modes: interactive, with a PTY, and command, without a PTY (e.g. a single command with separate output and error streams). Parameters: §14.6.

## 14.3	Opening
Only the Client opens a terminal Channel. A Server MUST NOT open one; a Client receiving it MUST close the Connection with PROTOCOL_VIOLATION.
```
terminal-open = [
  type: CHANNEL_OPEN,
  body: {
    0 => 1,                     ; channel kind: terminal
    1 => session-id / null,     ; null: create a new Session; session-id: attach (§15.1)
    * int => any,               ; terminal parameters: §14.6
  },
]

terminal-attached = [
  type: TERMINAL_ATTACHED,      ; value 6 (§11.3)
  body: {
    0 => session-id,            ; the attached Session (§15.1)
    1 => attachment-id,         ; this Attachment (§15.9)
    2 => uint,                  ; output offset of the first output byte that follows (§15.11)
    ? 3 => bstr,                ; terminal modes to set before the replay (§15.11)
    * int => any,
  },
]
```
- The Server checks the Authentication Context and authorizes the create or attach (§15.9). On success its first message on the stream is TERMINAL_ATTACHED; on failure it rejects the Channel (§8.3). An unknown Session ID is rejected with SESSION_NOT_FOUND (§15.2).
- The Client MUST NOT send any further message on the stream before TERMINAL_ATTACHED.
- A Connection MUST NOT have two terminal Channels attached to the same Session; the Server rejects the second CHANNEL_OPEN.
- Terminating a Session is not a Channel operation; it is a request on the control Channel (§15.6).

## 14.4	Stream
- A terminal Channel uses exactly one bidirectional stream. Data and control messages (input, output, resize, signals, exit status: §14.7–§14.10) travel in-band on it.
- Messages in each direction are processed in order. In particular, the exit status follows the last output, and a resize takes effect between the input bytes sent before and after it.
- Keystrokes are sent as they are typed; line editing and echo belong to the Server PTY.
- Values 1–23 are encoded by CBOR in one byte and are allocated only to frequent messages (terminal data, later file and forwarding data); others use 24 and above.

## 14.5	Number of Channels
- A Connection MAY have no terminal Channel (e.g. file transfer only, port forwarding only, Session list).
- The Server limits concurrent Channels per Connection with QUIC MAX_STREAMS (§8.2). RECOMMENDED default: 10.

## 14.6	Terminal parameters
```
terminal-open body, continued:
    ? 2 => pty-params,          ; present: create a PTY
    ? 3 => command,             ; absent: login shell of the Account
    ? 4 => { * tstr => tstr },  ; environment
    ? 5 => tstr,                ; Session name (§15.3)
    ? 6 => uint,                ; detach timeout in seconds (§15.8)
    ? 7 => uint,                ; attach mode: 0 shared (default), 1 takeover (§15.9)
    ? 8 => bool,                ; read-only Attachment (§15.9)
    ? 9 => uint,                ; resume: output offset of the first byte not yet received (§15.11)

pty-params = {
  0 => tstr,                    ; terminal type (TERM)
  1 => uint,                    ; columns
  2 => uint,                    ; rows
  ? 3 => uint,                  ; width in pixels
  ? 4 => uint,                  ; height in pixels
  * int => any,
}

command = tstr / [+ tstr]
```
Mode and command:
- PTY and command are independent: a login shell or a command, each with or without a PTY.
- A text string is a command line, run by the Account's shell (`-c` on Unix-like systems). An array is an argument vector: its elements reach the program unchanged, without splitting or expansion. A Client SHOULD use the array form when the arguments come from data rather than from a user typing a command line.
- Both forms run through the Account's shell, so a disabled or restricted shell (e.g. `nologin`, `git-shell`) also governs commands. On Unix-like systems the Server quotes each element of the array for a POSIX shell (in single quotes, a `'` as `'\''`), joins them with spaces and runs the result with `-c`.
- The command MUST NOT be empty and MUST NOT contain NUL. On a Server whose platform takes a single command line, the Server builds it from the array by that platform's quoting rules.

PTY:
- Terminal type: 1–64 bytes matching `[A-Za-z0-9][A-Za-z0-9._+-]*`. The Server sets `TERM` from this field only.
- Columns and rows: 1–65535. Pixel sizes: 0–65535; 0 means unknown.
- Terminal modes (termios) are not transmitted. The Server uses its defaults and enables UTF-8 input processing where the platform supports it (`IUTF8` on Linux).

Environment:
- Names are non-empty and contain neither `=` nor NUL; values contain no NUL. At most 64 variables; a name or a value at most 4096 bytes.
- The Server applies only variables on its allowlist and silently ignores the rest. The default allowlist is `LANG` and `LC_*`. `TERM` and `STPT_SESSION` (§15.4) are never taken from the environment field.
- A Client MUST NOT forward its whole environment; it sends only `LANG`, `LC_*` and variables configured explicitly for the Server.

Attach:
- When attaching to an existing Session (`1 => session-id`), only `pty-params` columns, rows and pixel sizes, the attach mode, read-only and resume are allowed; the size applies as a resize (§14.8, §15.9). Terminal type, command, environment, Session name and detach timeout MUST NOT be sent.
- Attach mode, read-only and resume MUST NOT be sent when creating a Session.

Validation:
- A malformed or out-of-range parameter, or a forbidden parameter on attach, rejects the Channel with CHANNEL_REJECTED; exceeded environment limits with LIMIT_EXCEEDED (§8.3, §13.2).

## 14.7	Terminal I/O
```
terminal-data = [
  type: TERMINAL_DATA,          ; value 7 (§11.3)
  body: { 0 => bstr .size (1..), * int => any },
]

terminal-stderr = [
  type: TERMINAL_STDERR,        ; value 8 (§11.3)
  body: { 0 => bstr .size (1..), * int => any },
]

terminal-eof = [
  type: TERMINAL_EOF,           ; value 24 (§11.3)
  body: { * int => any },
]
```
- TERMINAL_DATA carries input from the Client and output (stdout or PTY output) from the Server.
- TERMINAL_STDERR carries the error output of a command without a PTY; only the Server sends it, and only without a PTY.
- Data is an opaque byte string, not necessarily UTF-8. Message boundaries carry no meaning: a receiver MUST NOT assume that a message holds whole characters or lines.
- A sender SHOULD send data as soon as it is available, without delaying it to coalesce messages.
- TERMINAL_EOF ends the input of a command without a PTY; afterwards the Client MUST NOT send TERMINAL_DATA. With a PTY, end of input is the in-band EOF character (Ctrl-D), and TERMINAL_EOF MUST NOT be sent.
- The end of output is marked by the exit status, which follows the last output (§14.4, §14.10).
- Messages sent against these rules are a protocol violation (§0.2).

Backpressure:
- QUIC stream flow control is the backpressure. While the stream is blocked, the Server stops reading the PTY or the output pipes; it MUST NOT buffer output without bound.
- Output of a Session without an Attachment: §15.5.

Output is passed unmodified in both directions; escape sequences are not filtered (see Security Considerations).

## 14.8	Terminal resize
```
terminal-resize = [
  type: TERMINAL_RESIZE,        ; value 25 (§11.3)
  body: {
    1 => uint,                  ; columns
    2 => uint,                  ; rows
    ? 3 => uint,                ; width in pixels
    ? 4 => uint,                ; height in pixels
    * int => any,
  },
]
```
- Keys and ranges are those of `pty-params` (§14.6); the terminal type does not change after creation.
- Only the Client sends TERMINAL_RESIZE, and only on a Channel with a PTY; otherwise it is a protocol violation (§0.2). There is no reply.
- The Server applies the size to the PTY (e.g. `TIOCSWINSZ` on Unix-like systems, which delivers SIGWINCH to the foreground process group). The Server cannot change the size of the Client window.
- A Client SHOULD coalesce bursts of size changes and send only the latest size. A Server MAY apply only the last of several resizes queued without data between them, and MAY limit the rate at which it applies them.
- With one Attachment the size applies directly. The size given on attach (§14.6) is handled as a resize. Choice of size with several Attachments: §15.9.

## 14.9	Signals
```
terminal-signal = [
  type: TERMINAL_SIGNAL,        ; value 26 (§11.3)
  body: {
    0 => uint,                  ; signal id
    * int => any,
  },
]
```
| Id | Signal | Typical use |
|---|---|---|
| 0 | reserved; in TERMINAL_EXIT: a signal not in this table | |
| 1 | INT | interrupt |
| 2 | QUIT | interrupt with core dump |
| 3 | HUP | hangup, reload configuration |
| 4 | TERM | graceful termination |
| 5 | KILL | forced termination |
| 6 | USR1 | application-defined |
| 7 | USR2 | application-defined |
| 8 | ABRT | report only |
| 9 | SEGV | report only |
| 10 | PIPE | report only |
| 11 | BUS | report only |
| 12 | FPE | report only |
| 13 | ILL | report only |
| 14 | ALRM | report only |

Ids marked "report only" appear only in TERMINAL_EXIT (§14.10); sending them in TERMINAL_SIGNAL is a protocol violation.

- Signal ids are defined by this table, not by operating system numbers, which differ between platforms. The Server maps them to its platform. Further signals only through a capability (§11.4); an unknown id is a protocol violation (§0.2).
- With a PTY, the control characters (Ctrl-C, Ctrl-\, Ctrl-Z) are input bytes and the PTY generates the signals (§14.4); TERMINAL_SIGNAL is for explicit delivery. Without a PTY it is the only way: a Client SHOULD send INT when the user interrupts a command locally.
- Only the Client sends TERMINAL_SIGNAL, with or without a PTY, also after TERMINAL_EOF. There is no reply; if the target no longer exists or the Session is Exited, the signal is ignored.
- Target: with a PTY, the foreground process group of the PTY; without a PTY, the process group of the command. The Server starts a command without a PTY in its own process group.
- The Server MUST deliver a signal with the permissions of the Account: the result is the same as `kill(2)` called by that Account. A signal MUST NOT reach a process the Account could not signal itself.
- A Server on a platform without POSIX signals maps them where an equivalent exists (e.g. INT to a console control event, TERM and KILL to process termination) and ignores the rest.

## 14.10	Exit status and Channel close
```
terminal-exit = [
  type: TERMINAL_EXIT,          ; value 27 (§11.3)
  body: {
    ? 0 => uint,                ; exit code
    ? 1 => uint,                ; signal id (§14.9), if terminated by a signal
    ? 2 => bool,                ; core dumped
    * int => any,
  },
]
```
- Exactly one of keys 0 and 1 is present. The exit code is up to 2^32−1 (0–255 on Unix-like systems, 32-bit on Windows).
- A signal without an id in §14.9 is reported as id 0. A Client treats an id it does not know as id 0.

Sending:
- When the primary process (login shell or command) exits, the Session becomes Exited (§3.2). The Server reads the remaining output until end of file, but no longer than a grace period (RECOMMENDED 1 s, configurable), then sends TERMINAL_EXIT. Later output is discarded. Processes left running: §15.7.
- TERMINAL_EXIT is the last message of the Server on the stream; the Server then ends its direction (FIN).
- On attach to an Exited Session the Server sends the replay of its last output (§15.11), then TERMINAL_EXIT.
- A Session whose exit status was delivered to an Attachment becomes Terminated (§15.7).

Close:
| Situation | Behaviour |
|---|---|
| process exited | Server: TERMINAL_EXIT, FIN; Client: FIN |
| detach | Client: FIN; Server: FIN without TERMINAL_EXIT; the Session keeps running |
| detach by the Server (takeover, detach request, §15.9) | Server: FIN without TERMINAL_EXIT; Client: FIN |
| stalled stream (§15.9) | Server: RESET_STREAM with TIMEOUT |
| error | RESET_STREAM / STOP_SENDING with an error code (§13.1) |
| Session terminated (§15.6) | processes end; then as "process exited" |

- FIN from the Client means detach, not end of input (TERMINAL_EOF, §14.7). Detach is allowed with and without a PTY.
- Messages of the Client that crossed TERMINAL_EXIT are ignored silently; they are not a protocol violation.
- Every close ends the Attachment and never changes the Session state by itself (§3.4).

# 15	Session model

## 15.1	Session ID
```
session-id = bstr .size 16
```
- The Server generates the Session ID at creation: 16 bytes from a cryptographically secure random number generator. A Client never chooses it.
- IDs are unique per Server and never reused (§3.4). Random IDs keep this true across Server restarts without persistent state, so a Client holding an old ID cannot reach a new Session that got the same number; they also do not reveal how many Sessions exist.
- Text form: 32 lowercase hexadecimal digits, e.g. `3f2a9c0e5b7d41e6a8c2f09d1e4b7a53`. A Client MAY accept a unique prefix from the user and resolve it through the Session list (§15.10); on the wire the full ID is always sent.

## 15.2	Ownership
- A Session is owned by the Account of the Authentication Context that created it. The owner is fixed at creation and never changes.
- Every Connection authenticated for the owner Account, with any key authorized for it, may see and attach to the Session (attach rules: §15.9). Processes of one Account can reach each other's terminals at the operating-system level anyway, so a split by key inside the protocol would not be a security boundary. Restrictions within an Account: Tier PI.
- The Session ID is not a secret and grants no access: access is decided by the Account only. The ID may be displayed, logged and passed into the Session (§15.4).
- A Session of another Account is not accessible through the protocol, whatever the privileges of the Account on the Server.
- The Server MUST handle the ID of a Session of another Account exactly as an unknown ID: SESSION_NOT_FOUND in both cases, so the existence of other Accounts' Sessions is not revealed. The Server SHOULD make the response time independent of the reason.
- The Server keeps the User Identity that created a Session as metadata (Session list §15.10, audit Tier PH); it is not used for authorization.

## 15.3	Session name
- Names are for users; the protocol addresses Sessions by ID only. A Client resolves a name to an ID through the Session list (§15.10).
- A Client MAY give a name when it creates a Session (key 5 of terminal-open, §14.6). In this version a name is set only at creation; a Session without a name has none.
- Name: 1–64 bytes matching `[A-Za-z0-9][A-Za-z0-9._-]*`, compared byte by byte.
- A name is unique among the owner Account's Sessions that are not Terminated. A taken name rejects the Channel with SESSION_NAME_IN_USE; a malformed one with CHANNEL_REJECTED (§14.6).

## 15.4	Session ID inside the Session
- The Server SHOULD set the environment variable `STPT_SESSION` to the text form of the Session ID (§15.1) for the primary process of the Session, for `stpt` commands run inside it.
- The variable grants no rights (§15.2). It is never taken from the environment field of terminal-open (§14.6).

## 15.5	Creation and detach
- A Session is created by opening a terminal Channel without a Session ID (§14.3); there is no separate create request. It is Running once its primary process has started. If the start fails, the Channel is rejected and the Session is Terminated before its ID was ever sent.
- A Session that runs without an Attachment from the start is Client behaviour: the Client opens the Channel and ends it with FIN after TERMINAL_ATTACHED.
- Detach is the Client FIN on the terminal Channel (§14.10); a closed Connection detaches as well. Other forms of detach: §15.9.
- While a Session has no Attachment, the Server MUST keep reading its output, so its processes never block on a full PTY or pipe. Output is discarded except what the output buffer keeps (§15.11).

## 15.6	Termination
```
session-terminate = [
  type: SESSION_TERMINATE,      ; value 28 (§11.3)
  body: {
    0 => request-id,            ; §8.4
    1 => session-id,
    ? 2 => bool,                ; force: KILL at once
    * int => any,
  },
]
```
- Only the owner Account may terminate a Session (§15.2); any other request is answered with SESSION_NOT_FOUND, as are unknown and Terminated Sessions.
- Running: the Session becomes Terminating. The Server signals, with the permissions of the Account (§14.9), the processes of the Session: with a PTY HUP to all process groups of the PTY's terminal session, as a closed terminal would; without a PTY TERM to the process group of the command. Processes still running after the termination grace period (RECOMMENDED 5 s, configurable) receive KILL. With force set, KILL is sent at once.
- Processes that left the terminal session or process group (e.g. by `setsid`, daemons) are not part of the Session and are not signalled.
- When the processes have ended, the Session is Terminated: Attachments receive TERMINAL_EXIT and are closed as in §14.10, and the Server answers success. A process that cannot end after KILL (e.g. in an uninterruptible kernel wait) does not delay the result.
- Terminating: a further request waits for the same end; with force set it sends KILL at once.
- Exited: the Session is removed (Terminated) at once.

## 15.7	Exit and removal
- When the primary process exits, the Session becomes Exited (§14.10). The Server does not signal processes left running (e.g. started with `nohup` or in the background), neither at exit nor at removal; once the Session is Terminated, the Server closes its PTY and they are subject to the platform rules for a closed terminal.
- An Exited Session becomes Terminated as soon as an Attachment has received TERMINAL_EXIT and ended its direction of the stream (FIN): the Client FIN confirms the delivery of the exit status.
- If the Connection is lost before that FIN, or the Session had no Attachment when it exited, it stays Exited until a later Attachment confirms the exit status (§14.10), a termination request (§15.6), or Retention Policy (§15.8).

## 15.8	Retention Policy and limits
The Server alone sets Retention Policy (§4). RECOMMENDED defaults:
| Session | Default |
|---|---|
| Running without an Attachment | no limit; the Server MAY set a maximum time, after which it terminates the Session as in §15.6 |
| Exited, exit status not delivered | removed after 24 h |
| Sessions of one Account, not Terminated | at most 64; a further create is rejected with LIMIT_EXCEEDED |

Detach timeout:
- A Client MAY request at creation (key 6 of terminal-open, §14.6) how long the Session may stay without an Attachment, in seconds. The Server applies the smaller of this value and its own policy. The Client cannot extend the Server policy.
- The time runs from the end of the last Attachment and restarts with every new detach. When it expires, a Running Session is terminated as in §15.6 and an Exited Session is removed.
- 0: the Session ends as soon as it has no Attachment, also after a lost Connection.
- In this version the detach timeout is set only at creation.

- Sessions end when the Server stops; this version does not preserve them across a Server restart. Their IDs are then unknown (SESSION_NOT_FOUND) and never reused (§15.1).

## 15.9	Attachments
```
attachment-id = uint

session-detach = [
  type: SESSION_DETACH,         ; value 30 (§11.3)
  body: {
    0 => request-id,            ; §8.4
    1 => session-id,
    ? 2 => attachment-id,       ; absent: all Attachments
    * int => any,
  },
]
```
Authorization:
- Only Connections authenticated for the owner Account attach (§15.2). Attach is possible to a Running or Exited Session; a Terminating, Terminated, unknown or foreign Session is answered with SESSION_NOT_FOUND.
- Restrictions by key within an Account: Tier PI.

Attachment ID:
- The Server assigns an Attachment ID at create and at attach and returns it in TERMINAL_ATTACHED (§14.3). It is unique within the Session and never reused in it. Attachments of a Session are listed with the Session (§15.10).

Several Attachments:
- A Session MAY have several Attachments, at most one per Connection (§14.3). The Server limits them per Session (RECOMMENDED 8) and rejects a further attach with LIMIT_EXCEEDED.
- Attach mode (key 7 of terminal-open): shared (0, default) adds the Attachment to the existing ones; takeover (1) first ends all other Attachments of the Session as a detach by the Server (§14.10), then attaches.
- The Server sends output to every Attachment. Input and signals of all Attachments that are not read-only are applied in the order they reach the Server; there is no order between Attachments. TERMINAL_EOF from any of them ends the input of the command.

Read-only:
- A read-only Attachment (key 8 of terminal-open) receives output only. Its Client MUST NOT send TERMINAL_DATA, TERMINAL_EOF, TERMINAL_RESIZE or TERMINAL_SIGNAL; doing so is a protocol violation.
- Read-only is a convenience against accidental input, not a security boundary: the same Account may attach with full access.

Terminal size:
- The PTY has the size of the last active Attachment: the one that most recently attached, sent TERMINAL_DATA or sent TERMINAL_RESIZE. The Server keeps the last size of every Attachment and applies the size of the active one when it changes (§14.8).
- A read-only Attachment never becomes active, and its size is never applied.

Stalled Attachment:
- Output proceeds at the pace of the slowest Attachment (§14.7). If the Server cannot send on the stream of an Attachment because of flow control for longer than the stall limit (RECOMMENDED 10 s, configurable), it ends that Attachment with RESET_STREAM and TIMEOUT. The other Attachments and the Session continue, so a stalled Client (e.g. a suspended process) never blocks the processes of the Session.
- This applies to a single Attachment as well. A slow but progressing stream is not stalled.

Detach request:
- SESSION_DETACH ends one Attachment of the Session, given by its ID, or all of them. The detached Clients see a detach by the Server (§14.10). The Server answers once those Attachments have ended.
- Only the owner Account may send it (§15.2). An unknown Session is answered with SESSION_NOT_FOUND, an unknown Attachment ID with ATTACHMENT_NOT_FOUND.
- There is no request to detach "all but the current one": it would be a command from inside the Session controlling Attachments (DECISIONS.md §8). The same intent is expressed at attach by the takeover mode.

## 15.10	Session list
```
session-list = [
  type: SESSION_LIST,           ; value 31 (§11.3)
  body: {
    0 => request-id,            ; §8.4
    ? 1 => session-id,          ; only this Session
    ? 2 => session-id,          ; after: continue after this ID
    * int => any,
  },
]

session-list-result = REQUEST_RESULT body (§8.4), continued:
    2 => [* session-info],
    ? 3 => bool,                ; more: further Sessions follow

session-info = {
  0 => session-id,
  ? 1 => tstr,                  ; name (§15.3)
  2 => uint,                    ; state: 1 Running, 2 Exited, 3 Terminating
  3 => uint,                    ; creation time
  4 => bstr .size 32,           ; Ed25519 public key that created the Session
  ? 5 => tstr,                  ; label of that key
  6 => bool,                    ; PTY
  ? 7 => tstr,                  ; command for display; absent: login shell
  ? 8 => exit-status,           ; Exited only
  ? 9 => uint,                  ; removal time (Retention Policy, detach timeout)
  10 => [* attachment-info],
  * int => any,
}

attachment-info = {
  0 => attachment-id,           ; §15.9
  1 => uint,                    ; attach time
  2 => bstr .size 32,           ; Ed25519 public key of the attached Connection
  ? 3 => tstr,                  ; label of that key
  4 => bool,                    ; read-only
  5 => bool,                    ; active: its size applies (§15.9)
  ? 6 => address,
  * int => any,
}

exit-status = { ? 0 => uint, ? 1 => uint, ? 2 => bool, * int => any }   ; as the TERMINAL_EXIT body (§14.10)
address = [bstr .size 4 / bstr .size 16, uint]                          ; IPv4 or IPv6 address, UDP port
```
Scope:
- The list holds only Sessions of the Account of the Connection (§15.2) in the states Running, Exited and Terminating. With key 1 it holds only that Session; an unknown or foreign one is answered with SESSION_NOT_FOUND.
- Listing changes no state. In particular it does not deliver the exit status of an Exited Session (§15.7).
- A Client resolves Session names and unique ID prefixes from the list (§15.1, §15.3).

Paging:
- Sessions are ordered by Session ID, bytewise ascending. With `after` the list continues with the IDs greater than it; that Session need not exist any more.
- The Server puts as many Sessions as fit the maximum message size of the Client (§13.4) and sets `more` if others follow. Pages are not a snapshot: Sessions may appear or end between requests.

Fields:
- Times are Unix time in seconds (UTC) by the Server clock; the Client converts them for display.
- Public keys are sent as keys; the Client shows their fingerprints (§9.2).
- Label: an optional name the Server configuration gives to an authorized key (e.g. the comment of the key entry, such as `laptop-work`), 1–64 bytes. A Client MAY also keep its own local names for fingerprints.
- Command: for display only, never to be run. A text string is shown as it is, an argument vector joined with spaces; the Server cuts it to at most 256 bytes at a character boundary.
- Removal time: present while a deadline of Retention Policy or of the detach timeout applies to the Session (§15.8).
- Address: the Client address and UDP port of the Attachment's Connection as the Server sees them, the current ones after a migration. The Server MAY omit it by configuration.
- Limits on names, labels, command and Attachments (§15.3, §15.9) keep one session-info well below the minimum message size (§13.4).

## 15.11	Output buffer and replay
Output buffer:
- The Server keeps the most recent output of every Session in a buffer, with and without Attachments, until the Session is Terminated. RECOMMENDED size 256 KiB, configurable; 0 disables replay. Without a PTY it keeps standard output and error output in their order, each with its kind.
- Output offset: output bytes are numbered from 0 at Session start, both kinds together, in the order the Server reads them. Output discarded after the exit grace period (§14.10) is not numbered.
- A Client tracks the offset: the value of key 2 of TERMINAL_ATTACHED plus the bytes of every TERMINAL_DATA and TERMINAL_STDERR it has received since.

Replay on attach:
- After TERMINAL_ATTACHED the Server first sends replay from the buffer, then live output, without a gap or overlap between them. Key 2 of TERMINAL_ATTACHED is the offset of the first byte that follows. Replay uses TERMINAL_DATA and TERMINAL_STDERR as the original output did.
- Resume (key 9 of terminal-open present): replay starts at the given offset if the buffer still holds it, otherwise at the start of the buffer; the Client learns the lost amount from key 2 and SHOULD tell the user. An offset beyond the current end of output rejects the Channel with CHANNEL_REJECTED.
- Fresh attach (key 9 absent): replay is the tail of the buffer, at most the fresh replay limit (RECOMMENDED 16 KiB, configurable), starting after the first line feed in that tail, so it does not start inside an escape sequence; without a line feed in the tail nothing is replayed. With a PTY, if the current screen began within the tail (the last switch of the alternate screen, see Terminal modes; offset 0 if there was none), the replay starts exactly there instead: the output before belongs to another screen, and the start of a screen is not inside an escape sequence.
- Redraw: on a fresh attach, and on a resume that lost output, the Server sends SIGWINCH to the foreground process group of the PTY even if the size has not changed, so full-screen programs redraw the screen. Not on a resume without loss, which continues the screen as it was. As many programs redraw completely only on a real size change, the Server MAY instead change the PTY size by one row for a moment and restore it; each change delivers SIGWINCH.
- Exited Session: the same replay, then TERMINAL_EXIT (§14.10).
- The Server does not interpret output: it keeps no screen state and runs no terminal emulator; the only exception are the terminal modes below.

Terminal modes:
- With a PTY, the Server follows in the output the switches of a fixed set of terminal modes, which full-screen programs set once at their start and a new Client's terminal would otherwise lack: DEC private modes switched with `CSI ? Pm h` and `CSI ? Pm l` — cursor keys (1), mouse reporting (9, 1000, 1002, 1003, 1005, 1006, 1015), cursor visibility (25), alternate screen (47, 1047, 1049), focus events (1004), bracketed paste (2004) — and the keypad mode (`ESC =`, `ESC >`). Other sequences are not followed.
- On attach the Server puts into key 3 of TERMINAL_ATTACHED the sequence that sets every followed mode that is not at its default (alternate screen last); key 3 is absent if all are at their default. The Client writes it to its terminal before the replay, except on a resume without loss: that terminal already has the modes of the offset it resumes from, and the replay carries the switches since. It is not output: it has no output offset.
- A Client SHOULD follow the same modes in the output and return them to their defaults when its Attachment ends without TERMINAL_EXIT (detach, lost Connection), so its terminal is usable again.

## 15.12	Connection migration and reattach
- A network change of the Client (e.g. Wi-Fi to mobile) is QUIC connection migration (RFC 9000 §9): the Connection and its Attachments continue; nothing happens at the STPT level (§6.1). This holds while the outage is shorter than the idle timeout (§13.3).
- A Client SHOULD notice a change of its address and migrate at once with a new connection ID (RFC 9000 §9.5): until a packet from the new address arrives, the Server keeps sending to the old one, and an idle Client may stay silent until its next keep-alive.
- After the loss of a Connection the Client opens a new Connection, authenticates fully (§12; resumption never restores authentication, §6.2) and attaches to the Session with resume (§15.11). There is no resume token that would grant access without the user key.
- The old Attachment stays until the Server notices the loss (idle timeout or stall limit, §13.3). The Client MAY end it at once with SESSION_DETACH and its old Attachment ID (§15.9). It SHOULD NOT use the takeover mode for this, which would also end the Attachments of other devices.
- When to reattach automatically, how often to retry and how to show it to the user is Client behaviour. A failed verification of the Server key (§9.3) MUST NOT be retried or bypassed, and a Client MUST NOT ask the user to trust a new key while reattaching.
- Terminal input has no offsets: what was sent on the lost Connection may be lost. A Client SHOULD NOT reattach automatically when losing input would go unnoticed, such as standard input from a pipe, and SHOULD NOT send input typed during the outage later.

# 16	File transfer Channel

Status: draft file transfer profile 1 (PE001–PE005). The program implements the Channel envelope (RE001), POSIX Server file operations and ordinary verified transfers (RE002), including base path and mandatory kind/size metadata. Explicit upload/download resume (RE003), Windows drive encoding/Client parsing and informational POSIX metadata (RE004) are implemented on available platforms. Windows terminal Server is implemented (RC008, current Account/token); native Windows local-volume file operations, verified resume and metadata are implemented in program 0.4.1-0 (G3 resolved); Tier RE scenario review is complete in program 0.4.1-2; docs/RE_REVIEW.md records validation scope and fixes. Implementation limitations are recorded in DECISIONS.md §20–§26 and the program architecture; no full profile-1 conformance is claimed.

## 16.1	Purpose and boundaries
- A file transfer Channel provides Client-initiated transfer and management of files on the Server, with the permissions of the authenticated Account. Upload and download use the same Channel kind.
- The Channel belongs to its Connection, independently of any terminal Session or Attachment. Opening it neither creates nor attaches to a Session. No terminal is required.
- The Server MUST NOT gain access to Client files merely because this Channel exists. The Client chooses local sources and destinations; Server-initiated access to Client files is a separate service (Tier PF).
- Operation requests and outcomes are defined in §16.5–§16.9 (PE002), transfer data and integrity checks in §16.10–§16.13 (PE003), resume in §16.14–§16.17 (PE004), and paths and metadata in §16.18–§16.20 (PE005).

## 16.2	Negotiation and opening
Allocations: capability id 1 (`file-transfer`), Channel kind 2 (`files`), message types 0x4000–0x4007 (§11.3). These are capability messages, not core messages. This optional extension does not change the current wire version.

The Client offers capability 1 in CLIENT_HELLO with these parameters:
```
file-transfer-params = {
  0 => 1,                       ; profile version
  1 => uint,                    ; maximum simultaneous file Channels, at least 1
  * int => any,
}
```
The Client SHOULD offer a maximum of 1 by default and MAY allow a larger configured value. A Server supporting profile 1 MAY select capability 1 with the same profile version and a maximum between 1 and the offered maximum, capped by its configured per-Connection file-Channel limit (RECOMMENDED default: 8). Unsupported or malformed offers are not selected. A Client MUST reject a selection with an unsupported profile, missing required fields, a zero maximum or a maximum above its offer as PROTOCOL_VIOLATION.

The selected maximum is fixed for the Connection. The capability is in effect only after selection (§11.2). Absence of selection leaves terminal and Session services unaffected.

Only the Client opens a file transfer Channel, after authentication and only when this capability is in effect (§8.2, §11.4):
```
files-open = [
  type: CHANNEL_OPEN,
  body: {
    0 => 2,                     ; channel kind: files
    * int => any,
  },
]

files-ready = [
  type: FILES_READY,            ; value 0x4000; capability 1
  body: { * int => any },
]
```
- Opening carries no path, operation or Session ID. The Server checks the Authentication Context, file-service policy and resource availability. Acceptance grants no permissions beyond that Account's file access and any stricter Server policy.
- On acceptance the Server's first message is FILES_READY. The Client MUST NOT send further messages on this stream before receiving it. A repeated CHANNEL_OPEN or FILES_READY is a protocol violation.
- The Client SHOULD bound the wait for FILES_READY (RECOMMENDED 10 s from sending CHANNEL_OPEN); on expiry it resets the stream and sends STOP_SENDING with TIMEOUT. The Connection remains open.
- Rejection uses RESET_STREAM and STOP_SENDING with CHANNEL_REJECTED, or LIMIT_EXCEEDED for a resource limit (§13); the Connection and other Channels remain open.
- A Server MUST NOT open this Channel on the Client. Receipt of one, or use of the Channel without the negotiated capability, closes the Connection with PROTOCOL_VIOLATION.

## 16.3	Stream and concurrency
- Each file transfer Channel uses exactly one client-initiated bidirectional QUIC stream, using the framing, CBOR and peer message-size limits of §7 and §13.4. No raw byte mode or extra data stream is introduced.
- After FILES_READY the Channel carries only messages defined for this file-transfer profile. The Server MUST perform file operations with the Account's permissions, never with additional privileges of the Server process.
- At most one operation is outstanding on a Channel. A Client may reuse an idle Channel for subsequent operations after receiving FILE_RESULT (§16.5); parallel operations use separate Channels, opened as needed up to the selected maximum (§16.2).
- The Client MUST NOT exceed the selected maximum, counting Channels from sending CHANNEL_OPEN until both stream directions have ended or been aborted. The Server MUST enforce the maximum and rejects an excess opening with LIMIT_EXCEEDED without affecting other Channels. A lower currently available QUIC stream budget can further restrict parallelism; selection does not reserve streams.
- File Channels share the Connection's stream budget with terminal Channels (§14.5); opening a file Channel does not reserve an unlimited set of streams. Implementations MUST bound buffered file data and open file resources; QUIC flow control MUST apply to file data. Per-operation limits belong to PE002–PE003.
- A sender SHOULD prioritize control and interactive terminal traffic over bulk file data. Independent streams do not provide independent Connection congestion control.

## 16.4	Lifecycle and interruption
- Channel states: Opening → Ready → Closed; rejection goes from Opening to Closed. An operation can run only in Ready. Closed is final.
- A Client closes an idle Channel with FIN; the Server releases its Channel resources and finishes its direction with FIN. A Server ending an idle Channel similarly sends FIN; the Client finishes its direction. FIN alone MUST NOT be interpreted as successful completion of a file operation.
- An interrupted operation (FIN before its required result, reset or Connection loss) has an unknown outcome unless the operation rules establish otherwise. In particular, a Client MUST NOT blindly retry a potentially completed modifying operation. Publication and cleanup rules are in §16.8 and §16.16. Channel closure alone promises no rollback of an already committed operation.
- Cancellation or a local Channel failure uses RESET_STREAM and STOP_SENDING (§13). Resources owned only by the Channel are released; other Channels and terminal Sessions continue. Invalid protocol messages remain Connection errors (§0.2, §11.4).
- QUIC migration preserves the Channel. Connection loss ends it. A new Connection requires full authentication, capability negotiation and a new Channel. Resume requires an explicit request on a new file Channel (§16.14).

## 16.5	Operation exchange
All messages below travel on the file Channel, never on the control stream. Path and entry encodings are in §16.18–§16.19.
```
file-request = [
  type: FILE_REQUEST,             ; 0x4001, Client -> Server
  body: {
    0 => uint,                    ; operation ID
    1 => uint,                    ; operation kind (§16.6)
    2 => file-path,               ; source or target path
    ? 3 => file-path,             ; required for rename only; destination
    ? 4 => bool,                  ; replace, write/rename only; default false
    ? 5 => file-size,             ; required for write only (§16.10)
    ? 6 => file-hash-id,          ; read/write only, default 1 (§16.10)
    ? 7 => file-resume-request,   ; resume or opt in to upload retention (§16.14)
    * int => any,
  },
]
file-result = [
  type: FILE_RESULT,              ; 0x4002, Server -> Client
  body: {
    0 => uint,                    ; operation ID
    1 => uint,                    ; file status (§16.7)
    ? 2 => file-metadata,         ; successful stat only
    * int => any,
  },
]
file-entries = [
  type: FILE_ENTRIES,             ; 0x4003, Server -> Client
  body: {
    0 => uint,                    ; operation ID
    1 => [+ file-entry],          ; names and metadata (§16.19)
    * int => any,
  },
]
file-accepted = [
  type: FILE_ACCEPTED,            ; 0x4004, Server -> Client
  body: {
    0 => uint,                    ; operation ID
    1 => file-size,               ; exact transfer length (§16.10)
    2 => file-hash-id,            ; accepted hash algorithm (§16.10)
    ? 3 => file-resume-accepted,  ; required when request field 7 is present
    * int => any,
  },
]
```
- The Client chooses strictly increasing operation IDs on each Channel, starting at 0. IDs are not durable transaction IDs and have no meaning on another Channel. Before an ID would overflow the CBOR unsigned integer range, the Client opens a new Channel.
- After FILES_READY the Client sends one FILE_REQUEST and waits for its final FILE_RESULT before starting the next operation. The Server sends exactly one FILE_RESULT per request unless the Channel is interrupted. Receipt of a second outstanding request, an incorrect operation ID, a duplicate result, or a message in the wrong direction or phase is PROTOCOL_VIOLATION.
- `stat`, `mkdir`, `remove` and `rename` receive only FILE_RESULT. `list` receives zero or more FILE_ENTRIES followed by FILE_RESULT. Only `read` and `write` receive FILE_ACCEPTED, at most once, followed by the transfer exchange in §16.10–§16.12 and a final FILE_RESULT. A refusal before acceptance is a FILE_RESULT with nonzero status and no transfer exchange.
- FILE_ACCEPTED means that the Server has opened the source or prepared a private staging file and permits the transfer phase; it is not completion or a promise that the transfer will succeed. The Client MUST NOT send upload data before it receives FILE_ACCEPTED.
- Status 0 is final success. A nonzero status ends the operation as a failure, without closing the Channel; reuse is permitted only after the synchronization in §16.12. A failure that cannot be synchronized ends the Channel with a stream error instead. Neither FIN nor FILE_ACCEPTED substitutes for FILE_RESULT.
- A successful `stat` result MUST contain field 2. All other results, including failed `stat` results, MUST omit it. Required result fields are validated even for an unknown nonzero status.
- Defined fields used for another operation kind, a missing required field, or an incorrect field type are protocol violations. A well-formed request with an unknown operation kind receives UNSUPPORTED, with no filesystem effects. Unknown extension fields follow §11.4.
- A `rename` request MUST contain field 3; all other known operation kinds MUST omit it. Field 4 is permitted only for `write` and `rename`.

## 16.6	Operations
| Kind | Name | Meaning |
|---|---|---|
| 1 | read | Download one regular file. No implicit directory traversal or archive conversion. |
| 2 | write | Upload one regular file to a destination path; publish as specified in §16.8. |
| 3 | stat | Return metadata of the named directory entry, without following a final symbolic link. |
| 4 | list | Enumerate the immediate entries of one directory. |
| 5 | mkdir | Create one directory; its parent must exist. An existing entry is ALREADY_EXISTS. |
| 6 | remove | Remove one non-directory entry, or one empty directory. |
| 7 | rename | Atomically move one directory entry to the destination in field 3. |

- Paths are Server paths interpreted under §16.18. No operation executes a shell, expands wildcards, interprets environment variables or creates missing parent directories. Recursive copying or removal is not a primitive in this profile. A Client must request individual operations explicitly.
- Path traversal uses the Account's filesystem permissions and any stricter Server policy (§16.9). On POSIX, `read` and `list` MAY follow final symbolic links to a regular file and a directory respectively. Windows reparse points follow §16.18. Devices, FIFOs and sockets MUST NOT be read or written as file transfers.
- `stat`, `remove` and the source of `rename` operate on the final directory entry itself: a final symbolic link is inspected, removed or moved, not its target. `remove` never recurses and returns NOT_EMPTY for a nonempty directory.
- `write` defaults to create-only. With `replace=true`, an existing destination MUST be a regular file. Replacement publishes a new file object. Existing hard links and already-open handles can still refer to the old object. New object metadata follows §16.20.
- `rename` defaults to no replacement. With `replace=true`, compatible existing entries may be replaced according to the Server filesystem's rename rules; a replaced directory must be empty. A destination symbolic link is replaced as an entry, never followed. The rename MUST be atomic; if this is unavailable (including a cross-filesystem move), return UNSUPPORTED, never fall back to copy-and-delete. A rename resolving to the same entry may succeed without a change if the filesystem supports it.
- Replacement is an explicit Client request, not a Server prompt. The Server MUST enforce create-only/no-replace at the filesystem mutation itself, not through an earlier existence check followed by an unconditional replacement. If the platform cannot enforce the requested semantics it returns UNSUPPORTED.
- `list` uses bounded FILE_ENTRIES messages that fit the Client's message-size limit. It MUST NOT collect an unbounded directory in memory. Names represent immediate entries (§16.19). The Server MUST omit the special entries `.` and `..` if its native enumeration returns them. An empty directory produces no FILE_ENTRIES, only FILE_RESULT(OK). Order is unspecified. Concurrent changes can affect enumeration. Data before a failed or interrupted final result is an incomplete listing.
- A read is not a snapshot either. Concurrent modification can change the bytes observed; §16.11 defines byte counts and integrity verification without claiming a consistent filesystem snapshot. Separate Channels have no shared operation ordering or transaction.

## 16.7	File statuses
FILE_RESULT uses its own status namespace, not the QUIC error-code namespace of §13.2. No operating-system error number or diagnostic text is sent.

| Status | Name | Meaning |
|---|---|---|
| 0 | OK | operation completed |
| 1 | NOT_FOUND | a required path component or entry is absent |
| 2 | ACCESS_DENIED | Account permissions or Server policy refuse the operation |
| 3 | ALREADY_EXISTS | destination exists and replacement is not permitted |
| 4 | NOT_DIRECTORY | a required directory is not a directory |
| 5 | IS_DIRECTORY | an operation requiring a non-directory encountered a directory |
| 6 | NOT_EMPTY | directory is not empty |
| 7 | INVALID_PATH | path is invalid or not representable on the Server platform |
| 8 | NO_SPACE | storage space or filesystem quota exhausted |
| 9 | READ_ONLY | target filesystem is read-only |
| 10 | UNSUPPORTED | operation or required semantics are unsupported |
| 11 | LIMIT_EXCEEDED | configured resource limit exceeded |
| 12 | IO_ERROR | other filesystem or local I/O failure |
| 13 | INTEGRITY_ERROR | transfer verification failed (§16.11) |
| 14 | RESUME_UNAVAILABLE | saved transfer cannot be resumed (§16.17) |
| 15 | RESUME_BUSY | matching upload may still be active (§16.17) |

- Unknown nonzero statuses are failures equivalent to IO_ERROR, never success. A response with invalid CBOR or invalid field types remains a protocol violation.
- The Server maps native errors to these statuses without exposing privileged path information. Policy may return ACCESS_DENIED instead of revealing whether an inaccessible entry exists; the Server MUST NOT probe it with elevated credentials to produce a more specific status.
- A failure result does not imply that any bytes already downloaded can be trusted, that a listing is complete, or that arbitrary native filesystem failures have transactional rollback. The stronger upload publication rules are in §16.8. Detailed diagnostics belong in the Server log.

## 16.8	Upload publication and interruption
- A write stages data in a newly created private regular file on the destination filesystem. Staging MUST NOT truncate or modify an existing destination. The Server MUST prevent other Accounts from reading or changing staged data and MUST NOT reuse an attacker-controlled staging entry. A failed setup receives a failure result before FILE_ACCEPTED.
- Publication occurs only after the complete transfer and successful verification defined by PE003. It atomically installs the staged file at the requested destination with the requested replacement policy and the Account's permissions. If publication cannot preserve these guarantees, the operation fails rather than exposing a partial destination.
- OK is sent only after publication succeeds. Success means logical filesystem completion, not guaranteed persistence after power loss; no storage durability barrier is promised by this profile.
- Failure before publication leaves an existing destination untouched by this operation and does not install the new destination. The Server removes unpublished staging data on failure or cancellation except for an eligible interruption under §16.16. Cleanup failures MUST be logged, and remaining staging data MUST retain its access protections. Retention never exposes partial data at the final path.
- Cancellation is a stream reset (§16.4), not a rollback request. It can race with publication. If publication succeeded but FILE_RESULT was lost, the Client cannot distinguish success from failure; it MUST report an unknown outcome and MUST NOT automatically repeat a modifying operation. The same uncertainty applies to `mkdir`, `remove` and `rename` after loss of their result. There is no exactly-once guarantee across Channels or Connections.
- Failure or interruption of a download leaves local destination and partial-file handling to the Client. A Client MUST NOT report a successful download without successful transfer verification and the final OK result. Local file overwrite and cleanup require the Client's own explicit destination policy, not instructions from the Server.

## 16.9	Authorization and resource boundaries
- Every filesystem access, including path resolution, metadata lookup, directory enumeration, staging, publication and cleanup, MUST use no more authority than the authenticated Account and the Server's file-service policy allow. Server process privileges MUST NOT bypass filesystem permissions, ACLs, quotas or read-only restrictions.
- Authentication or Channel acceptance is not a cached permission grant for future operations. The Server MUST enforce authorization at each actual access, using the platform's filesystem checks and race-safe filesystem operations. A path check followed by access with elevated credentials is insufficient. Permissions and directory entries may change between operations.
- File access does not require a Session or a usable interactive shell. Server policy MAY deny file service independently of terminal access. The base profile provides normal Account file access, not an implicit jail within the Account's home; a restricted-root policy requires confinement that remains effective through symbolic links and concurrent path changes.
- Implementations MUST bound open descriptors, staging bytes, pending listing data and concurrent work. The Server MAY refuse work with LIMIT_EXCEEDED or a stream error when it cannot continue safely. No recursive traversal, executable file content inspection or shell invocation is needed to service these operations.

## 16.10	Transfer messages and sizes
Profile 1 transfers file data sequentially, starting at offset 0 for a fresh transfer and at the verified offset for resume (§16.14). The data sender is the Client for `write`, the Server for `read`; the other Endpoint is the receiver.
```
file-size = 0..2305843009213693951 ; 2^61 - 1 bytes, common size bound
file-hash-id = uint

file-data = [
  type: FILE_DATA,               ; 0x4005, data sender -> receiver
  body: {
    0 => uint,                  ; operation ID
    1 => file-size,             ; byte offset
    2 => bstr .size (1..61440),  ; file bytes, at most 60 KiB
    * int => any,
  },
]
file-end = [
  type: FILE_END,                ; 0x4006, data sender -> receiver
  body: {
    0 => uint,                  ; operation ID
    1 => uint,                  ; sender status (§16.7)
    2 => file-size,             ; start offset plus bytes sent as FILE_DATA
    ? 3 => bstr .size 32,       ; accepted algorithm's digest, required iff status is OK
    * int => any,
  },
]
file-received = [
  type: FILE_RECEIVED,           ; 0x4007, Client -> Server, read only
  body: {
    0 => uint,                  ; operation ID
    1 => uint,                  ; receiver status (§16.7)
    * int => any,
  },
]
```
- A write request MUST contain field 5 with the exact number of bytes to upload; other operation kinds MUST omit it. FILE_ACCEPTED field 1 echoes that length for a write; the Client MUST reject a different length as PROTOCOL_VIOLATION. For a read the Server obtains the length from the opened regular file before acceptance. The accepted length is fixed for this operation.
- A read or write request MAY set field 6 to one algorithm ID. Omission selects BLAKE3 (ID 1). Other operation kinds MUST omit field 6. Profile 1 implementations MUST support BLAKE3 and MAY support SHA-256 (ID 2). The Client MUST support any algorithm it requests.
- The Server confirms the effective algorithm in FILE_ACCEPTED field 2. If the requested ID is unknown or unavailable, it replies with FILE_RESULT(UNSUPPORTED) before acceptance or transfer data. The Client MUST reject a different accepted ID as PROTOCOL_VIOLATION and MUST NOT automatically retry with another algorithm after a rejection. The accepted ID applies to the entire transfer and any resumed attempt (§16.14).
- The common length bound is 2^61 - 1 bytes for either algorithm, keeping optional SHA-256 input below 2^64 bits. Implementations MUST use checked integer arithmetic for offsets, lengths and their sums, without truncation to a native word size. Platform or policy limits may be smaller. A valid request beyond a local limit receives LIMIT_EXCEEDED or UNSUPPORTED. Files larger than 4 GiB require no different encoding or additional stream.
- FILE_DATA is permitted only after FILE_ACCEPTED, with the current operation ID, in the data sender's direction and before FILE_END. Offsets MUST equal the start offset plus preceding data lengths. Gaps, overlap, empty data, offset overflow or data beyond the accepted length are PROTOCOL_VIOLATION. Once the offset reaches the accepted length, only FILE_END may follow in this transfer phase.
- A sender SHOULD use chunks of 32 KiB by default; chunk boundaries have no file semantics. The 60 KiB maximum leaves room for required CBOR fields within the minimum message-size limit. The entire encoded message, including extension fields, MUST still fit the peer's announced limit (§7.2). No padding to a fixed chunk size is required.
- An empty file uses no FILE_DATA messages, followed by FILE_END with length 0 and, on success, the selected algorithm's digest of the empty byte sequence. Zero-length data is not an end marker. No compression, sparse-hole instruction, out-of-order block, or per-block acknowledgement is defined in this profile. Sparse files transfer their logical bytes, including zeroes.

## 16.11	Integrity and source consistency
- A successful transfer MUST use the algorithm confirmed in FILE_ACCEPTED, implemented by a proven cryptographic library. ID 1 is [BLAKE3](https://github.com/BLAKE3-team/BLAKE3-specs) in unkeyed hash mode with its standard 32-byte output. ID 2 is SHA-256 as specified in [FIPS 180-4](https://www.nist.gov/publications/secure-hash-standard). FILE_END carries the 32 raw digest bytes. The digest covers the retained prefix, if any, followed by the ordered concatenation of FILE_DATA byte strings. CBOR, offsets, metadata, block boundaries and other messages are outside the hash input.
- The sender initializes the hash with the verified prefix on resume and updates it incrementally with the data sent. It sends exactly one FILE_END after its last data message. On success it sends OK with the accepted length and digest. On source failure it sends a nonzero file status and the start offset plus the number of bytes sent, with no digest. No data may follow FILE_END for that operation.
- The receiver independently counts and hashes the retained prefix and received data. For a successful FILE_END it MUST verify that the declared count, observed count including the prefix, and accepted length agree and that the digest matches before reporting success or publishing an upload. A mismatching count or digest is INTEGRITY_ERROR. A missing or incorrectly typed required field remains PROTOCOL_VIOLATION. A failure FILE_END also MUST report the observed count including the prefix. A mismatch is INTEGRITY_ERROR. A nonzero sender status can never become success.
- For uploads the Server hashes the bytes it successfully writes to staging; a local write error prevents success. Any buffered writes and close errors relevant to publication MUST be checked before publication and OK. For downloads the Client similarly checks local writes, buffered completion and relevant close errors before acknowledging successful receipt. This verification detects a transfer mismatch; it does not prove that storage will retain the data, require a read-back of the destination, or replace TLS peer authentication and integrity.
- A read sends exactly the initial accepted length from the opened file: appended bytes are not included. An EOF before that length is a source failure (IO_ERROR), not a successful shorter file. A write Client applies the same rule to its local source and declared length. A sender MAY fail with IO_ERROR if it detects a source change while reading. Neither the hash nor the initial length guarantees a snapshot: concurrent in-place changes may produce a mixture of versions with a valid digest of the bytes actually transferred.
- The receiver MUST NOT allocate a file-sized buffer or trust an unverified digest or length to authorize filesystem effects. Hash and byte counts are maintained with bounded memory. This specification promises neither a consistent snapshot of a mutable source nor cryptographic protection against an authenticated sender deliberately choosing different contents.

## 16.12	Completion and failure synchronization
Successful upload:
```
Client                                      Server
  |-- FILE_REQUEST(write, length, [hash ID]) ->|
  |<----------- FILE_ACCEPTED(length, hash ID)|
  |-- FILE_DATA* ---------------------------->|
  |-- FILE_END(OK, length, digest) ----------->|
  |             verify, close and publish     |
  |<------------------------- FILE_RESULT(OK)|
```
- The Server MUST NOT send an upload FILE_RESULT after acceptance until it has consumed FILE_END. A count mismatch, or a digest mismatch in a successful FILE_END, returns INTEGRITY_ERROR even if the sender also reports a failure. Otherwise, on sender failure it returns that failure status (unknown nonzero statuses map to IO_ERROR); on sender success it verifies local completion and publishes under §16.8, then returns OK or the publication error. Any failure prevents publication. FILE_RECEIVED is not used for uploads.
- If the Server cannot continue receiving an upload safely (e.g. storage exhaustion, permission loss, or staging failure before FILE_END), it MUST abort the Channel with RESET_STREAM and STOP_SENDING rather than send an early FILE_RESULT or drain an unbounded remainder. Use LIMIT_EXCEEDED for a resource limit, INTERNAL_ERROR for other local failures. File statuses and QUIC error codes are separate namespaces. The Client monitors the receive direction while uploading and stops producing data when the Channel is aborted.

Successful download:
```
Client                                      Server
  |-- FILE_REQUEST(read, [hash ID]) ---------->|
  |<----------- FILE_ACCEPTED(length, hash ID)|
  |<----------------------------- FILE_DATA* |
  |<----------- FILE_END(OK, length, digest)  |
  | verify and finish local writes           |
  |-- FILE_RECEIVED(OK) --------------------->|
  |<------------------------- FILE_RESULT(OK)|
```
- After receiving FILE_END the Client MUST send exactly one FILE_RECEIVED, after consuming all preceding data and checking the source status, count, digest and local writes. A count mismatch, or a digest mismatch in a successful FILE_END, takes precedence and is reported as INTEGRITY_ERROR. Otherwise, on sender failure it echoes the nonzero sender status (unknown statuses map to IO_ERROR); on a local destination failure it reports its corresponding nonzero file status. It MUST NOT acknowledge OK unless all these checks succeeded.
- The Server waits for FILE_RECEIVED before sending the download FILE_RESULT. It returns INTEGRITY_ERROR if the receipt reports that status; otherwise, if its FILE_END status was nonzero, it returns that sender status; otherwise it returns the receipt status. Unknown nonzero statuses map to IO_ERROR. A download receipt before FILE_END was sent, multiple receipts or an upload receipt is PROTOCOL_VIOLATION. The Server MUST monitor incoming messages during sending, not wait until the entire file has been queued before reading its receive direction.
- A Client unable to continue receiving a download MAY abort the Channel immediately instead of draining the remaining data. On local failure, the sender may likewise abort if it cannot send a well-formed FILE_END at a frame boundary. Aborting uses §16.4 and leaves other Channels and Sessions intact. An Endpoint that started a frame MUST complete that frame before sending another message; it cannot insert FILE_END into a partly written FILE_DATA frame.
- After FILE_END the sender sends no more transfer data. A final FILE_RESULT proves that the Server consumed the upload end marker, or the Client's download receipt; the Client may then start its next operation with no old transfer messages outstanding. Verification failures after a complete transfer therefore permit Channel reuse. A reset, FIN or lost Connection at any earlier point is interruption, not successful synchronization or completion.
- If the Client cannot finish the receipt exchange or if the final result is lost, it MUST NOT report successful completion. An upload may already have been published (§16.8); a verified local download may exist but the protocol operation has not completed. Resetting a Channel does not revoke a publication already committed. No automatic retry or resumption is introduced here.

## 16.13	Flow control and progress limits
- QUIC supplies reliable ordered delivery and flow control. There are no STPT retransmissions or individual chunk acknowledgements. A sender MUST bound its pending data queues and a receiver MUST apply backpressure rather than buffering an entire file. File length, chunk size and Channel parallelism do not reserve memory or disk space.
- Both Endpoints MUST remain able to process stream aborts, control traffic and local cancellation while sending, receiving or performing filesystem work. A sender SHOULD give interactive and control traffic priority (§16.3). Parallelism uses additional negotiated Channels, not additional streams for one transfer.
- Implementations SHOULD impose a configurable no-progress deadline (RECOMMENDED 60 s) from sending or receiving FILE_REQUEST through the final FILE_RESULT, including setup, directory enumeration, resume-prefix verification, data transfer and completion. Progress is completion of operation messages or actual local filesystem, I/O or verification work, not keep-alive PINGs or unrelated Connection activity. Deadline expiry aborts only this Channel with TIMEOUT. This bounds stalled operations even if QUIC remains alive; it is not a total operation duration limit. A peer cannot observe the other Endpoint's local work and may time out while that work is still in progress; long prefix verification may therefore require a larger configured deadline. Such an abort follows the usual interruption and unknown-outcome rules.
- A receiver MUST enforce its configured disk, descriptor and staging limits while work is in progress. An advertised length is not a storage guarantee. Cancellation and lost results follow §16.8 and §16.17. Resume validation and retention are defined in §16.14–§16.17.

## 16.14	Resume requests
Resume is optional in profile 1, with upload retention and resumed downloads supported independently. FILE_REQUEST field 7 is explicit. A Server that does not support the requested mode returns UNSUPPORTED before acceptance. The Client starts a resumed operation on a new authenticated file Channel, with an operation ID local to that Channel. It MUST verify the same Server Identity and use the original Account. It MUST use the original remote path (§16.18), hash algorithm, transfer length and, for uploads, replacement policy.

```
file-resume-token = bstr .size 32
file-prefix-digest = bstr .size 32

file-resume-request =
    [0]                                           ; fresh write, retain on interruption
  / [1, file-resume-token]                        ; resume write
  / [2, file-size, file-size, file-prefix-digest]  ; resume read: total length, offset, digest
  / [3..18446744073709551615, * any]              ; unknown modes receive UNSUPPORTED

file-resume-accepted =
    [file-resume-token, file-size, file-prefix-digest] ; write: token, offset, digest
  / [file-size]                                       ; read: verified offset
```

- Modes 0 and 1 are write only. Mode 2 is read only. A fresh read needs no advance opt-in. Other operations MUST omit field 7. A known mode with invalid fields or direction is PROTOCOL_VIOLATION. An unknown mode identified by an unsigned integer in the first array element receives UNSUPPORTED. A resume read offset MUST be greater than zero and no greater than its supplied total length.
- If a request includes field 7 and is accepted, FILE_ACCEPTED MUST include field 3 in the form for that operation. All other acceptances MUST omit field 3. An accepted offset MUST be within the confirmed total length. A fresh retained upload starts at zero and confirms the digest of the empty byte sequence. A resumed upload echoes the same token. A resumed download echoes the exact requested offset and length. A missing, unexpected, malformed or inconsistent acceptance is PROTOCOL_VIOLATION.
- A prefix is the contiguous byte range from zero up to but excluding the accepted offset. Its digest uses the transfer's confirmed algorithm. Each side rereads its local prefix in bounded buffers and feeds the same bytes into both prefix verification and the full-file hash, for example by finalizing a copy of the hash context. The prefix digest itself MUST NOT be used as input in place of the prefix. Importing serialized library hash state over the wire is outside this profile.
- After prefix verification, FILE_DATA begins at the accepted offset and transfers the suffix. FILE_END reports the logical count including the prefix. Its success digest covers the whole logical file (§16.11). The accepted offset may equal total length. In that case the sender sends FILE_END with no FILE_DATA and completes the usual result exchange.

## 16.15	Download resume
- The Client retains a private partial destination and the original Server Identity, Account, remote path, total length and algorithm. It only checkpoints bytes successfully written from complete, contiguous FILE_DATA messages. It verifies its saved offset against the local partial file and recomputes the prefix digest before requesting resume. Unconfirmed bytes past the checkpoint MUST be discarded before appending. The Client MUST prevent changes to the retained prefix by other local operations under its control during resume.
- The Server holds no download resume record. It opens the source under current authorization and checks that its length equals the supplied original length. It hashes the prefix through the same open file handle it will read the suffix from. If the length or prefix does not match, it returns RESUME_UNAVAILABLE without FILE_ACCEPTED or FILE_DATA. On a match it sends FILE_ACCEPTED and continues from that offset. The Client keeps its prefix hash context and verifies FILE_END before acknowledging successful receipt.
- This verifies that the retained prefix matches the source bytes reread during resume. The suffix is read from the source during the resumed attempt. A changed suffix may be accepted, and concurrent changes still carry the limits of §16.11. A Client MUST NOT present this as restoration of an immutable historical version or silently overwrite a local destination with unverified output.

## 16.16	Upload checkpoints and publication
- The Server generates an opaque, unpredictable token using a proven cryptographic random number generator before accepting a write in mode 0. It binds the token to the Server Identity, authenticated Account, destination path, replacement policy, total length, algorithm and private staging object. The token is a lookup key and does not grant filesystem authority. Tokens MUST be protected from disclosure, including in logs and diagnostics.
- The saved record MUST also identify the destination parent directory resolved during the initial write setup, so a resumed attempt can enforce the parent-identity check in §16.18. A path string alone is not sufficient evidence of directory identity.
- The Server permits at most one active writer per token across all Channels and Connections. On an eligible interruption it checkpoints a contiguous prefix from complete FILE_DATA messages written to staging. It may roll back to an earlier complete message boundary or to zero. It MUST trim any bytes past that checkpoint before reuse. The offset counts saved bytes. QUIC acknowledgements do not establish a checkpoint. The Client MUST use the offset the Server confirms.
- Only detected Connection loss or a no-progress timeout before a complete FILE_END has been received can make an upload eligible for retention. A peer may signal a no-progress timeout by aborting the Channel with TIMEOUT. For explicit cancellation the Client aborts with NO_ERROR (§13.2). If the Server observes explicit cancellation, graceful shutdown, malformed messages, local transfer failure or a completed FILE_END, it clears the resumable record. If Connection loss hides a cancellation from the Server, the retained prefix is still subject to the Server's retention limits. Retained prefixes MAY be evicted earlier to enforce quotas.
- On mode 1 the Server rechecks Account access and file-service policy, verifies the bound arguments, acquires exclusive access to the staging object and recomputes its prefix digest. It sends the token, offset and digest in FILE_ACCEPTED. The Client compares with the prefix of its local source. The source length MUST still match the original length. On a local mismatch after acceptance the Client MUST abort the Channel as a cancellation and MUST NOT send further transfer data. On a match it sends the suffix from the same open source and computes the full-file digest.
- Processing a complete FILE_END makes the token permanently ineligible for resume before any publication attempt. This transition MUST be serialized with acquiring the token for resume. Publication still requires the full digest check and the same permissions and atomic replacement rules as a fresh write (§16.8). The Server MUST enforce authorization again at publication. A resumed upload cannot obtain permission from the original acceptance or its token.
- The Server MUST bound the number, total staging bytes and retention time of saved uploads, with per-Account and total limits. A recommended idle retention time is 1 hour from the last interruption. The Server removes expired, evicted and invalidated staging objects, subject to the cleanup rules in §16.8. Upload resume records are scoped to a running Server instance. After a restart old tokens MUST be rejected and abandoned staging MUST be cleaned up safely without publishing it. The Client may enforce its own local retention limits for partial downloads.

## 16.17	Refusal, retry and lost results
- RESUME_UNAVAILABLE covers expired, evicted, invalid, consumed or mismatched resume state and a changed download length or prefix. The Server uses the same response for an unknown upload token and for one bound to another Account or mismatched arguments. It MUST NOT reveal offsets or the existence of a record to a different Account. A policy or filesystem refusal MAY instead result in ACCESS_DENIED after the request is independently known to be authorized to inspect that resume state.
- RESUME_BUSY applies to a matching upload attempt whose earlier Channel still owns the token, including while the Server has yet to detect the interruption. The Client MAY retry that same resume request with bounded backoff. RESUME_BUSY never authorizes creating a fresh write. A request exceeding resource limits may receive LIMIT_EXCEEDED.
- If the Client can establish that no complete FILE_END was sent for a write, it MAY automatically resume that same retained upload, using the original token and normal authorization checks. If it cannot establish this and no final result arrived, the publication outcome is unknown (§16.8). The Client MUST report the uncertainty and MUST NOT automatically resume or submit a fresh write. A later explicitly requested attempt with the original token can only continue an eligible, unpublished checkpoint. RESUME_UNAVAILABLE does not say whether publication occurred.
- A refused resume does not fall back to an operation from offset zero or to a different algorithm. A fresh transfer requires a separate Client decision. PE004 provides no durable receipt or exactly-once guarantee across Server restarts. Successful resumed transfers use the same FILE_END, FILE_RECEIVED (read only) and FILE_RESULT exchange as fresh transfers.

## 16.18	Path representation and resolution
```
posix-name = bstr
windows-name = tstr
file-path =
    [0, [* posix-name]]                           ; absolute POSIX path
  / [1, tstr, [* windows-name]]                   ; absolute Windows drive path
```

- Paths identify objects on the Server. For kind 0, components are the exact POSIX name bytes. For example, `/tmp/a` is `[0, [h'746d70', h'61']]`. The empty array denotes the filesystem root. Each component MUST exclude NUL and slash. Backslash is an ordinary byte. Names can contain non-UTF-8 bytes and MUST be preserved exactly.
- For kind 1, the second element is one uppercase ASCII drive letter, without a colon. For example, `C:\Data\a` is `[1, "C", ["Data", "a"]]`. An empty component array denotes that drive's root. Names are Unicode scalar text encoded as CBOR text strings. A Client may normalize a user-entered drive letter to uppercase before constructing the wire path. The Server MUST use the native Unicode filesystem interface and MUST NOT perform lossy encoding conversion.
- Components MUST NOT be empty, `.` or `..`. Paths are absolute and have no current-directory, current-drive, home-directory or Session-relative interpretation. Root paths are valid for `stat` and `list` only. The protocol does not case-fold or normalize any name. Native filesystem lookup determines whether different spellings name the same entry.
- The Windows path kind addresses local volume drives only. UNC paths, network-mapped drives, drive-relative and root-relative paths, device and extended namespaces, and alternate data streams are outside this profile. Windows components MUST exclude U+0000 through U+001F and the characters `< > : " / \ | ? *`. They MUST NOT end in an ASCII space or dot. Reserved DOS device names MUST be rejected, including with an extension and irrespective of ASCII case. Reject `CON`, `PRN`, `AUX`, `NUL`, `COM1`–`COM9`, `LPT1`–`LPT9` and the superscript digit variants specified by Win32. Match the device name against the stem before its first dot after trimming trailing ASCII spaces. A Server MUST reject any path that its native API would interpret as a device or namespace alias. See [Win32 naming rules](https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file).
- A Server MUST support the path kind for its filesystem interface and MUST reply with UNSUPPORTED if it receives the other kind and does not implement it. A recognized path with invalid component contents, a bad drive letter, an invalid root operation or a native length limit violation receives INVALID_PATH. An invalid array shape or CBOR field type is PROTOCOL_VIOLATION. Encoded messages remain subject to §7.2.
- A Windows Server MUST NOT traverse a reparse point, in any component, in profile 1. It may inspect a final reparse point as an entry for `stat` or report it in a parent listing. It MAY remove or rename a final symbolic link or junction if the native operation acts on the link entry itself. Replacing a final reparse point by `write` is forbidden. Operations requiring traversal or unsupported mutation return UNSUPPORTED. Servers MUST perform authorization and these checks without a race between validation and use (§16.9). See [Win32 reparse points](https://learn.microsoft.com/en-us/windows/win32/fileio/reparse-points).
- The Client MUST retain the original path value for resume. Upload tokens bind the exact path kind, drive when present, and ordered component values. An alias spelling is a mismatch even if it resolves to the same entry. If the verified destination parent now resolves to a different directory from the original upload, resume returns RESUME_UNAVAILABLE. Publication MUST target the verified directory and apply current authorization and replacement rules (§16.8, §16.16).

## 16.19	Entries and metadata
```
file-entry = [
  posix-name / windows-name,                     ; one immediate entry name
  file-metadata,
]

file-time = [ -9223372036854775808..9223372036854775807, 0..999999999 ]
file-metadata = {
  0 => uint,                                     ; entry kind below
  ? 1 => uint,                                   ; logical size in bytes, regular files only
  ? 2 => file-time,                              ; modification time
  ? 3 => 0..511,                                 ; POSIX rwx mode bits
  ? 4 => bool,                                   ; Windows read-only attribute
  * int => any,
}
```

| Kind | Entry |
|---|---|
| 0 | Other or unknown |
| 1 | Regular file |
| 2 | Directory |
| 3 | Symbolic link, including a Windows junction |

- The Server MUST send field 1 for a regular file and omit it for other kinds. File size is the observed logical length, which may exceed the transfer limit. Field 2 is an optional UTC timestamp as whole seconds since 1970-01-01T00:00:00Z plus a nonnegative nanosecond fraction. For example, half a second before the epoch is `[-1, 500000000]`. A missing field means unavailable or not reported. An unknown entry kind is treated as other, with no assumptions about its attributes.
- Field 3 is optional and contains only the low nine POSIX permission bits. Field 4 is optional and describes the Windows read-only attribute. Each field applies only to the corresponding path kind. These values are informational. Access depends on the actual filesystem, ACLs and Server policy. Listing and stat do not expose owners, ACLs, link targets, security descriptors, extended attributes or native IDs in profile 1.
- Both `stat` and directory entries report metadata for the named object before any final link is followed. An unknown Windows reparse point is reported as kind 0. A listing name MUST use the matching path kind's component rules, including the name representation and validity checks. A Server that cannot encode an encountered entry, for example a Windows name with an unpaired surrogate, MUST end the list with UNSUPPORTED. If a single entry cannot fit the peer's message limit, the Server ends the list with LIMIT_EXCEEDED. Earlier batches remain an incomplete listing. An invalid listing name or metadata field for a known entry kind is PROTOCOL_VIOLATION.
- A name received from the Server is one remote component. For any local path derived from it, the Client MUST validate it as exactly one component under the local filesystem's rules, reject unrepresentable names without lossy substitution, and prevent escape through local links or reparse points. If different remote names would resolve to the same local destination, the Client MUST reject the collision or require explicit distinct local names. Terminal display MUST escape control characters in remote names.

## 16.20	Metadata on creation and replacement
- Metadata in §16.19 is read-only on the wire. A transfer carries file contents. Profile 1 does not copy source timestamps, permissions, owners, ACLs, attributes or special bits to the destination. A destination uses its own filesystem metadata and applicable Server or Client policy.
- The Server creates private upload staging under the Account's authority (§16.8). Publication MUST keep the uploaded object private to the Account under ordinary filesystem access. On POSIX the initial requested file mode is at most `0600`, with no executable or special bits. On Windows the Server MUST use an access descriptor and publication procedure that keep the object private to the Account under ordinary access, subject to system administration. If the platform or filesystem cannot provide this guarantee, the Server returns UNSUPPORTED.
- With `replace=true` a successful upload installs the new object's ownership, access controls and timestamps. It does not copy the old destination's metadata or broaden access through an inherited or retained ACL. `mkdir` uses native creation defaults under Account permissions and Server policy. `rename` moves an existing object with native rename metadata semantics. None of these operations accepts metadata overrides.

# 17	Client file access permissions (PF001, draft)

Implementation status: program 0.4.2-0 implements the internal Client grant lifecycle (RF001). Program 0.4.3-0 adds the Session-local command, protected origin and Attachment selection (RF002). Program 0.4.4-0 implements the Server-opened read/create exchange with explicit confirmation in a second local Client terminal (RF003). Program 0.4.5-0 implements the PF004 read-only mount on Linux with separate local consent and independent revocation (RF004).

## 17.1	Authority and consent

The Client is the enforcement point for access to its files. By default it MUST deny all Server-initiated file access, including discovery of names, existence and metadata. Authentication, capability negotiation, terminal input, opening a Channel and a previous §16 transfer MUST NOT confer permission. A Server request is a proposal only. The Client MUST refuse a request unless it can identify a single live local Attachment to a Running Session on the requesting authenticated Connection.

The Client MUST obtain explicit local user consent before issuing a grant. Its trusted local interface MUST identify the verified Server Identity, Account, Session and Attachment, the locally selected object, permitted operations and duration. It MUST distinguish remote suggestions from local selections and prevent remote output or terminal input forwarded to the Session from supplying or impersonating approval. Missing local interaction, dismissal and timeout MUST result in denial. Before granting access the Client MUST NOT probe a remote-suggested local path on the Server's behalf or disclose local path details in refusal responses.

The user authorizes disclosure or changes at the request of the Server. The Client cannot verify which remote process originated a request or keep disclosed data from the Server administrator. For directory mounts the Client MUST warn that Server root can read the exposed files and require an explicit local yes.

## 17.2	Scope and enforcement

A grant is private Client state bound to the verified Server Identity, authenticated Account, live Connection and exact Session and Attachment IDs. A reference to a grant MUST NOT serve as bearer authority. Each other Attachment, including one on another device belonging to the same Account, requires its own local consent. Requests MUST NOT be broadcast to find a willing Client.

The Client MUST bind a grant to a locally selected regular file, a new entry at an exact name in a selected parent, or an explicitly selected directory with stated descendant scope. It MUST grant only the operations shown to the user. Reading file contents, inspecting metadata, listing names, creating, modifying or replacing, deleting and renaming entries are separate rights. The local interface MAY offer clearly enumerated groups. A rename MUST cover both endpoints and any replacement. Expanding scope, rights or duration requires fresh consent. A one-operation approval MUST be consumed when the operation is admitted, including on failure, and MUST NOT authorize a retry. Repeated operations require an explicitly approved continuing grant.

For every operation the Client MUST check the live grant, scope, requested effects and applicable local policy, and act with the local user's filesystem authority. It MUST bind access to the approved object or parent and constrain descendant resolution during the actual operation. Path substitution, symlinks, reparse points, mount crossings and aliases MUST NOT redirect access beyond the approved scope. If the Client cannot enforce confinement or safely publish an approved change it MUST refuse. The Server needs only a grant-relative name or reference. The Client MUST NOT send local absolute paths as an incidental part of granting access.

## 17.3	Revocation and lifecycle

The Client MUST provide local revocation for continuing grants. Revocation, expiry, detach (including takeover), Session exit or termination, and detected Connection loss invalidate the affected grants and pending approvals. The Client MUST serialize invalidation with operation admission and publication. Once it observes invalidation it MUST refuse new work, stop further reads and sends, cancel outstanding work and discard unpublished changes. Data already sent and changes already committed cannot be recalled. Losing a grant MUST NOT terminate the terminal Session or affect another Attachment's grants.

Migration of the same authenticated Connection preserves the binding. Reconnect, automatic reattach, a fresh Attachment and Client restart MUST NOT restore it. Mounts MUST end when their grant ends.

## 17.4	Requests originating in a Session (PF002, draft)

A program running inside a Session submits its intended operation and an optional exact Attachment ID to a Server-local interface. The Server MUST establish the originating Running Session and its owner Account from a protected local context established for that Session. Possession of STPT_SESSION (§15.4), a claimed process ID, or an Account name alone is insufficient. A caller MUST NOT select another Session by supplying its ID. The Server MUST refuse calls for which it cannot establish the context. Local IPC and command syntax are implementation choices.

The local interface MUST let the caller discover the current Attachment IDs for its own Session and supply one as the target. A listing is a snapshot. In one selection step the Server consults the current Attachment registry for that Running Session:

| Caller selection | Server action |
|---|---|
| Explicit ID | Select only that live Attachment in this Session. An unknown or detached ID fails locally. |
| No ID, exactly one live Attachment | Select that Attachment. |
| No ID, several live Attachments | Return an ambiguous-target failure locally. The caller can list IDs and submit a new request with one ID. |
| No ID, no live Attachment | Return a no-target failure locally. |

Count all live Attachments before checking capability, terminal read-only mode or availability of a local confirmation interface. The Server MUST NOT infer which device supplied a shell command from keystrokes, most recent activity or attach order. Terminal read-only mode (§15.9) alone neither authorizes nor prevents a separate local file decision.

On selection, pin the tuple (authenticated Connection, owner Account, Session ID, Attachment ID) for this attempt. The Server MUST check its policy, confirm that the Connection negotiated the client-file-access service and revalidate the live binding before dispatch. Selection and pinning MUST use the same registry snapshot. A later join does not make an already selected target ambiguous. Revalidation tests that exact tuple, never the current count or an ID in another Session. Failure ends the attempt for the local caller. The Server MUST NOT queue it for a future attach, choose a fallback or notify other Clients. Capability 1 (§16) does not enable this service.

Deliver the proposal once, as the opening request of a dedicated Server-initiated bidirectional Channel (§8.3, PF003) on the selected Connection. The opening request MUST identify the exact Session and Attachment and describe the requested operation. Its stream correlates the Client response and subsequent exchange with the originating local call. No request is encoded in terminal output, replayed on attach or sent as a reverse §8.4 control request. PF003 defines negotiation, opening fields, responses and deadlines.

Before prompting and again before granting access, the Client MUST match both IDs to its own live Attachment on this authenticated Connection and apply §17.1–§17.3. A stale or mismatched target MUST be refused without prompting. The Server MUST cancel pending delivery or approval when the caller abandons the request, its deadline expires or the binding ends. The Client MUST dismiss a pending prompt on observed cancellation. Both endpoints MUST discard late approval for a cancelled attempt. Pending calls and prompts MUST be bounded and MUST NOT block terminal I/O. Report refusal, cancellation or loss to the originating caller. Delivery or consent alone MUST NOT be reported as operation success. A lost result after a modifying operation may have committed MUST be reported as an unknown outcome and MUST NOT trigger automatic retry.

## 17.5	Negotiation and opening (PF003, draft)

Capability 2 is optional and independent of capability 1. Offer and selection both use `{0 => 1, 1 => 1..65535, * int => any}` (profile, maximum concurrent kind-3 Channels per Connection). The Client offers it only when locally enabled. The Server may omit it or select profile 1 and a maximum no greater than offered. Ignore unsupported or malformed offers. An invalid selection is PROTOCOL_VIOLATION. The RECOMMENDED offered maximum is one. After authentication an enabled Client supplies QUIC credit for Server-initiated bidirectional streams (§8.2). Credit alone grants no permission. Both endpoints MUST count opening, prompting and active Channels toward the selected limit until completion or abort, independently of transport credit.

Only the authenticated Server on the Connection selected under §17.4 opens kind 3, once per proposal. The following frames use §7 framing and §7.3 unknown-key handling. Numeric message types are shown in wire order within each frame:

```
client-access-open = [3, {0 => 3, 1 => session-id, 2 => attachment-id,
                          3 => uint, ? 4 => file-size, * int => any}]
access-pending = [0x4010, {* int => any}]
access-approved = [0x4011, {* int => any}]
access-start = [0x4012, {* int => any}]
access-ready = [0x4013, {0 => file-size, * int => any}]
access-data = [0x4014, {0 => file-size, 1 => bstr .size (1..61440), * int => any}]
access-end = [0x4015, {0 => file-size, 1 => bstr .size 32, * int => any}]
access-result = [0x4016, {0 => uint, * int => any}]
access-denied = [0x4017, {* int => any}]
```

Open key 3 identifies the requested action: 1 reads one locally selected regular file, 2 creates one new locally selected regular-file destination. Only action 2 carries key 4, the exact proposed byte count. Neither action carries a pathname, overwrites an entry or grants directory access. Reject all other action IDs without prompting unless an extension selected on this Connection defines the action (§17.8). Additional rights or mount operations require a negotiated extension (PF004). Optional fields ignored under §7.3 MUST NOT expand authority.

## 17.6	Consent and transfer

1. The Client checks grammar, selected service, binding, local policy and prompt capacity before presenting anything. Refuse an unsupported action, stale target, disabled interface or policy failure using §8.3 CHANNEL_REJECTED. Abort excess Channels with LIMIT_EXCEEDED. A valid admitted proposal receives exactly one Client ACCESS_PENDING. This acknowledges admission only.
2. Use a trusted local interaction isolated from remote terminal rendering and input. Show the §17.1 context and direction in local terms (send one file to the Server, or save a new file of the stated size). The user selects the exact local object and explicitly approves that selection for one operation. No default affirmative action, forwarded keystroke, pasted remote output or terminal replay may approve. Serialize or independently identify simultaneous prompts. If isolation is unavailable, refuse. Denial, dismissal or inability to obtain consent after PENDING sends ACCESS_DENIED then Client FIN. The Server ends its send direction with FIN and reports refusal. The refusal contains no local diagnostic or filename.
3. Bind consent to the selected object (or selected parent and exact new name) at confirmation. Retain a safe reference or verify identity at actual use; a replacement object at the same pathname requires fresh consent. After rechecking the binding, approval sends ACCESS_APPROVED once. The Server checks that its caller, deadline and pinned target are still live before sending ACCESS_START. Before START the Client MUST NOT disclose file information or bytes, create the destination or consume the approval for another operation. On receiving START it rechecks the binding and grant, atomically admits this one operation (§17.2), and either sends a nonzero ACCESS_RESULT then FIN, or sends ACCESS_READY. READY declares the opened source length for read or echoes the proposed length for create. A different echoed length violates the protocol. Approval alone never means that the operation completed.
4. After READY, the data sender (Client for read, Server for create) sends zero or more ACCESS_DATA frames then ACCESS_END and FIN. Key 0 in DATA is the contiguous offset starting at zero. END gives the total count and raw BLAKE3 digest of all bytes, including the empty sequence. The receiver enforces the declared size, local limits, bounded buffering and full count/digest verification (§16.10–§16.13). A local sender failure aborts the stream. The Client stages a create privately and publishes only verified bytes, with atomic no-replace under current local authority. A read grant covers only the selected opened source.
5. After END the receiver sends ACCESS_RESULT then FIN. Status uses §16.7 (0 success, unknown nonzero means failure). Success requires verified receipt and successful local completion, including publication for create. For read the Server returns success to its caller only after its own verified local delivery. For create the Server requires the Client's complete success result. A receiver that must stop before END aborts the Channel. For a pre-READY failure in step 3 the Server replies with FIN. No stream reuse, resume or automatic retry is permitted.

For actions 1 and 2, an incorrectly typed or missing required field, key 4 on a read, a DATA offset gap or overlap, or DATA beyond the declared length violates the protocol. A well-formed END whose count or digest fails verification yields ACCESS_RESULT(INTEGRITY_ERROR) without publication. The digest covers the logical file bytes in order. For an empty file the sender sends END with count zero and the BLAKE3 digest of the empty sequence. A local failure reported with a stream abort uses INTERNAL_ERROR unless a more specific §13.2 code applies.

Example, creating an empty file: Server OPEN(action 2, size 0), Client PENDING then APPROVED, Server START, Client READY(0), Server END(0, empty digest) then FIN, Client RESULT(OK) then FIN. If another process creates the destination before publication the Client instead returns RESULT(ALREADY_EXISTS) and preserves that entry.

## 17.7	Cancellation, races and limits

An abort sends RESET_STREAM and STOP_SENDING on directions still open. Either side may cancel at any phase using NO_ERROR. Caller abandonment during a one-shot operation or mount setup, revocation or loss of the target binding requires cancellation. After mount handoff the setup caller no longer owns the operation (§17.8). Dismiss the prompt, release its approval and discard unpublished staging on observed abort or Connection loss. The Client MUST serialize a local cancellation with admission and publication (§17.3). A Server that cancels before sending START can report that no operation started. After sending START it MUST report an unknown modifying outcome if it lacks a definitive result and cannot rule out publication. Cancellation cannot undo an earlier commit. Serialize local cancellation with authorization to send START and with delivery of a final result to the caller. If cancellation wins, never send a new START or deliver a later success. If a definitive result was delivered first, a later abort does not change it. Ignore frames crossing a local cancellation on that stream, including late approval. After sending END and FIN an Endpoint MUST still monitor the peer direction for a result or abort; FIN alone does not cancel completed input. A FIN without the required preceding terminal frame ends an incomplete exchange as a Channel-local interruption, never success.

Use bounded monotonic timers on both endpoints. The Server opening deadline covers obtaining stream credit, sending OPEN and receiving PENDING. Client receipt of an incomplete opening frame is bounded from the first stream data. Consent time runs from sending PENDING at the Client and receiving it at the Server until approval or denial. On approval the Client starts its START deadline. The Server starts a no-progress deadline when sending START, and both endpoints enforce it during transfer and while waiting for the final result or peer FIN. Only progress in the awaited exchange counts. The RECOMMENDED defaults are in §13.3. A shorter caller deadline takes precedence. Expiry aborts only this Channel with TIMEOUT (or fails locally before stream allocation) and invalidates pending approval. Repeated frames, PINGs and prompt activity MUST NOT extend an opening, consent or START deadline.

The Client also bounds sending PENDING from receipt of OPEN by its opening deadline. While flushing a terminal response or awaiting the closing FIN, either endpoint applies the transfer no-progress deadline, including after denial or pre-READY failure.

Bound pending calls, prompts, staged bytes and active transfers locally, including across Connections. Continue servicing cancellation, deadlines and unrelated Channels during confirmation and I/O. Malformed frames, wrong initiator, unnegotiated use, wrong-direction or out-of-phase messages on a live exchange are PROTOCOL_VIOLATION (§0.2). A denial, native I/O failure, digest mismatch or expired grant is an ordinary Channel outcome and MUST NOT close the Connection.

## 17.8	Read-only directory mount (PF004, draft)

Capability 3 extends capability 2 on the same kind-3 Channel. Its offer and selection value is the unsigned integer 1 (read-only mount profile). The Client MUST offer it only alongside capability 2. The Server MUST ignore an unsupported or malformed offer and select it only together with capability 2 and a usable local mount backend. An invalid selection is PROTOCOL_VIOLATION. Each mount occupies one negotiated kind-3 slot for its entire lifetime.

The Server routes the request under §17.4 and opens action 3, omitting key 4. Without capability 3 the Client rejects this action without a prompt. For this action the local caller remains the owner of the request through setup. After successful installation the Session owns the mounted view subject to the original Attachment binding; normal exit of the setup command does not end it. A setup failure or abandonment before handoff aborts the Channel and removes any partial mount.

Use PENDING, APPROVED, DENIED and START with the directions and deadlines of §17.6–§17.7. Instead of one-shot approval, the Client requires the user to select one local directory in the trusted local interface and explicitly enter or select **yes**. Show the exact local directory, Server Identity, Account, Session, Attachment, and that the Server may repeatedly list names, inspect metadata and read regular files throughout the selected subtree until access ends. Explicitly warn that Server root can read and retain copies of those files. Explain that other Attachments of this Session can drive processes that read the mounted view. They cannot inherit or renew the grant. Show how to revoke the mount locally. Decline, dismissal, unavailable trusted UI or expiry grants no access. No saved confirmation, remote terminal input or approval for a previous mount satisfies this request.

Bind the directory identity at confirmation as in §17.6. On START, revalidate the grant and pin that same directory under the local user's authority; refuse if its identity cannot be established safely. Reply with a nonzero ACCESS_RESULT and FIN on setup failure, or MOUNT_READY on success. The latter starts the continuing grant. The Server serializes exposure and handoff with caller abandonment and binding invalidation. Revalidate the live binding in that transition. If invalidation wins, discard even a concurrently received MOUNT_READY, abort and dismantle any provisional backend without exposing it. If handoff wins, later binding loss revokes the installed view. It reports success to the local caller only when the backend is serving at a Server-selected mountpoint and returns that mountpoint locally. The mountpoint MUST be fresh, protected against substitution and private to the intended Session. The Server MUST refuse installation unless it can isolate the view from unrelated Sessions and ordinary Accounts, enforce read-only, no execution, no device or set-ID semantics, and tear down access independently of busy filesystem handles. Server administrators and processes inside the authorized Session remain within the exposure described by the prompt.

## 17.9	Mounted filesystem operations

```
mount-ready = [0x4018, {0 => 0..1, * int => any}]   ; name kind: POSIX or Windows (§16.18)
mount-call = [0x4019, {0 => uint, ? 1 => [* (bstr / tstr)], ? 2 => uint,
                       ? 3 => file-size, ? 4 => 1..61440, * int => any}]
mount-reply = [0x401A, {0 => uint, ? 1 => file-metadata, ? 2 => [* file-entry],
                        ? 3 => uint, ? 4 => bstr .size (0..61440), * int => any}]
```

After READY, only the Server sends CALL and only the Client sends REPLY, one outstanding call per stream, exactly one reply per call unless aborted. CALL key 0 selects the operation. REPLY key 0 is a §16.7 status. On failure omit all defined payload fields. On success use exactly the fields below. No request ID is needed. A second outstanding call, wrong phase, missing required or inapplicable defined field is PROTOCOL_VIOLATION. An unknown operation receives UNSUPPORTED.

| CALL operation | Required CALL fields | Successful REPLY |
|---|---|---|
| 1 stat | 1 relative path | 1 metadata (§16.19) |
| 2 list | 1 relative path | 2 complete array of immediate entries, omitting dot entries |
| 3 open regular file | 1 relative path | 3 fresh mount-local handle bound to the opened object |
| 4 read | 2 handle, 3 byte offset, 4 maximum bytes | 4 bytes at that offset, at most the requested count, empty at EOF |
| 5 close | 2 handle | No payload. Release the handle before replying. |

Paths start at the pinned directory. The empty array denotes the root. Every component uses the name kind in READY and the component rules in §16.18. No absolute path, drive prefix, parent step or platform translation may escape the root. Check authority and confinement at actual use (§17.2), including reads on held handles. If a moved object cannot still be verified as authorized, refuse further access. Never follow symlinks, reparse points or nested mounts or open special files. Stat and list may describe such entries without traversal or link-target disclosure. Enforce native naming without lossy conversion or collisions on the Server. If a name cannot be represented, fail the affected lookup or listing. All mutations and unsupported filesystem operations fail locally at the Server.

A listing need not be a snapshot across calls. Return LIMIT_EXCEEDED if a complete listing cannot fit one bounded reply. Never truncate it into apparent success. Bound enumeration memory, active mounts, handles and queued backend calls. Handles are scoped to this live Channel and never reused there. Unknown or closed handles receive NOT_FOUND. Concurrent native operations may queue within a bounded limit; no head-of-line wait on a mount may block terminal or control traffic. Reads observe the current opened file and provide no whole-file snapshot guarantee.

## 17.10	Revocation and unmount

Either endpoint may unmount at any time using the §17.7 stream abort with NO_ERROR. The Client MUST offer local revocation throughout. Each endpoint independently invalidates its side on explicit unmount, end of the bound Attachment (including takeover), Session exit or termination, detected Connection loss, Channel closure or local backend failure. The Server MUST initiate teardown when it observes such an event without waiting for the Client, and vice versa. Losing an unrelated Attachment does not end the mount. Migration on the same Connection preserves it. Reattach, reconnect and process restart require a fresh mount and consent.

Serialize invalidation with request admission and result delivery. Stop admitting work immediately on observed invalidation, fail pending filesystem requests and discard late replies. Both sides MUST recheck their local live mount state before handing a read or listing result to the transport or filesystem caller, including after asynchronous I/O. An invalidated instance MUST NOT accept callbacks or replies for a later mount at the same pathname. The Client releases all pinned objects even if CLOSE never arrives. The Server disables the backend, disconnects the filesystem and removes the mountpoint it owns once the OS permits. Open descriptors, cached pages, directory entries and busy working directories MUST NOT keep serving the revoked view. Disable caching, mappings or backend features where necessary to enforce this boundary. If native unmount is delayed, the view MUST already be inaccessible. Server shutdown or helper death must leave a disconnected view and release grants on detected stream loss. Previously read bytes, including copies in remote process memory, cannot be recalled.

Opening and consent use §17.7. While installing the mount or awaiting an operation reply, enforce the §13.3 no-progress deadline (recommended 60 s); abort this mount with TIMEOUT on expiry. Apply the progress deadline at both endpoints while servicing a CALL, including a partial request or reply and a blocked send. A healthy idle mount has no outstanding operation or partial frame and may remain idle. On a silent network partition, loss is observed through the QUIC idle timeout or an earlier outstanding-call deadline. No offline cache, background reconnect or automatic remount is permitted.

# 18 Port forwarding (PG001–PG004, draft)

## 18.1 Scope and lifetime
Profile 1 defines TCP local, remote and dynamic forwarding. It is an optional draft extension, implemented by program 0.5.0-1 (RG001–RG004; full scenario review). The CLI currently opens one rule per foreground command/Connection; runtime limits and validation scope are recorded in the program architecture and RG_REVIEW.md. It uses neither shell commands nor a terminal Channel and requires no Session. UDP, Unix-domain sockets, SOCKS BIND and Session-persistent listeners are outside this profile. UDP remains PG005 in the backlog.

A forwarding rule belongs to exactly one authenticated Connection and its Authentication Context. A kind-4 rule Channel holds its lifetime; each accepted TCP connection uses a separate kind-5 flow Channel on that Connection. Neither a rule nor an established TCP flow can move to another Connection. QUIC migration preserves both. Closing the Connection ends all rules, sockets and pending work, even if an unrelated Session survives. Terminal detach or Session exit alone does not end a rule. Reconnect requires new explicit local configuration and fresh authorization; it MUST NOT replay TCP bytes or resume an old flow. This Connection binding is the PG001 lifetime decision.

The Client's trusted local configuration authorizes the direction, listener exposure and targets for each rule. Remote terminal output or a Server-opened flow MUST NOT create or widen that configuration. In particular, a remote rule authorizes exactly one Client-side destination, never an address chosen by the Server. Approval through a CLI configuration is sufficient; a per-flow prompt is not required.

## 18.2 Negotiation and allocations
Capability 4, `tcp-forwarding`, assigns Channel kinds 4 (`forward-rule`) and 5 (`forward-flow`), and messages 0x4020 FORWARD_READY, 0x4021 FORWARD_CONNECTED and 0x4022 FORWARD_DATA. They do not change the base wire version. No forwarding message is a control-Channel request.

The Client offers:
```
tcp-forwarding-params = {
  0 => 1,                       ; profile
  1 => uint,                    ; simultaneous rules, 1..64
  2 => uint,                    ; simultaneous flows, 1..1024
  3 => [+ (1 / 2 / 3)],         ; distinct modes: local / remote / dynamic
  * int => any,
}
```
The Server selects the same profile, positive maxima no larger than the offer and a nonempty subset of the offered modes. Unsupported or malformed offers are not selected. A Client MUST treat a malformed selection, duplicate modes, an enlarged maximum or an unoffered mode as PROTOCOL_VIOLATION. RECOMMENDED selected maxima are 8 rules and 64 flows. These are aggregate per-Connection limits across all directions, including opening and teardown. When remote mode is selected, the Client MUST make server-initiated bidirectional stream credit available after authentication for authorized remote flows, subject to the shared Channel and resource limits (§8.2). Selection advertises protocol support, not Account authorization. The Server still checks each rule and flow after authentication.

Use without capability 4, use before authentication, a Server-opened rule Channel or a rule mode not selected in HELLO is PROTOCOL_VIOLATION. Only the Client opens rule Channels. Local and dynamic flow Channels are Client-opened; remote flow Channels are Server-opened. Before permitting a remote rule the Client MUST provide server-initiated bidirectional stream credit (§8.2). Limits and scheduling MUST reserve capacity for control and other enabled services; an accepted TCP socket MUST NOT wait indefinitely for QUIC stream credit. No Server-opened flow is authorized merely by stream credit.

## 18.3 Addresses
```
forward-ip = [1, bstr .size 4] / [2, bstr .size 16]
forward-host = forward-ip / [3, tstr]
forward-target = [forward-host, 1..65535]
forward-bind = [forward-ip, 0..65535]
forward-socket = [forward-ip, 1..65535]
```
IP byte strings use network byte order. Kind 3 is a DNS name: 1..253 ASCII bytes, lowercase labels of 1..63 characters, letters/digits with interior hyphens, separated by dots, no empty label or trailing dot. Unicode names require local conversion to ASCII A-labels before sending. Numeric addresses MUST use an IP form, not DNS text. IPv4-mapped IPv6 addresses MUST be normalized to IPv4 before sending and before policy comparison. A receiver MUST reject a noncanonical mapped representation as UNSUPPORTED without a network side effect. Scope identifiers, scoped IPv6 destinations, multicast and unspecified destinations are unsupported. DNS resolution is performed only by the Endpoint making the target TCP connection, subject to §18.8. There is no receiver-side shell, URL, service-name, search-suffix or proxy-environment interpretation.

Bind addresses are numeric only; unspecified addresses mean an explicit wildcard request. Port 0 requests an OS-assigned port. A receiver MUST NOT silently widen an address, use another fixed port or substitute a wildcard for loopback. IPv6 listeners MUST be IPv6-only so they cannot accidentally expose IPv4. Opening IPv4 and IPv6 listeners requires separate rules. RECOMMENDED local and remote listener defaults are an explicit loopback address and a user-selected port. A Client MUST require explicit local configuration for any non-loopback listener, including a wildcard; the Server must additionally authorize every remote bind. A well-formed address outside this profile returns UNSUPPORTED rather than attempting a fallback.

## 18.4 Rule creation and cancellation
```
forward-rule-open = [
  type: CHANNEL_OPEN,
  body: {
    0 => 4,
    1 => 1 / 2 / 3,             ; local / remote / dynamic
    ? 2 => forward-target,      ; required only for local
    ? 3 => forward-bind,        ; required only for remote
    * int => any,
  },
]
forward-ready = [
  type: FORWARD_READY,          ; 0x4020; Server to Client, once
  body: {
    0 => uint,                 ; status (§18.7)
    ? 1 => forward-socket,      ; required only for successful remote rule
    * int => any,
  },
]
```
The rule ID is the QUIC stream ID of its kind-4 Channel (an unsigned integer at most 2^62-1). It is scoped to this Connection, is never reused and grants no authority on another Connection. It is implicit in FORWARD_READY; no secret token or persistent registry is needed. Missing required known fields or known fields forbidden for the mode are protocol violations; unknown keys follow §7.3.

After authentication and negotiation, the Client sends CHANNEL_OPEN and waits for exactly one FORWARD_READY. Before acceptance the Server validates mode, Account policy, limits and parameters. For remote mode it binds/listens using the Account's permitted OS authority before sending success. FORWARD_READY reports the requested address and actual port; a nonzero requested port MUST match exactly; for wildcard listeners that address remains unspecified. The Client MUST validate the reply against its request and local configuration. Local and dynamic listeners are created/activated by the Client only after success, using locally configured addresses that are not sent on the wire. If local setup fails the Client cancels the rule. Local rules pin their target in field 2. Dynamic rules select a target per flow. A remote target remains solely in the Client's local rule object and is not sent to the Server.

A remote Server MUST NOT open flows until it has sent successful FORWARD_READY. Streams are independently ordered: a remote flow can reach the Client before READY. The Client MUST bound any such wait by the rule-opening deadline, validate against its own pending remote rule, and MUST NOT connect a target or acknowledge that flow before validating successful READY. Refused or cancelled rules cannot dispatch waiting flows.

For a valid, authorized kind, policy/bind/setup outcomes use FORWARD_READY. Non-success carries no field 1; the Server then sends FIN and the Client answers FIN without any further message. A failure creates no active rule. Generic admission refusal before this exchange MAY instead use CHANNEL_REJECTED or LIMIT_EXCEEDED (§8.3). After success neither Endpoint sends further messages on the rule stream. FIN, reset or STOP_SENDING in either direction revokes the entire rule; the other Endpoint closes its side without waiting for acknowledgement. FIN is cancellation, not TCP half-close, on this Channel.

Revocation MUST first invalidate the rule locally, stop accepting and close its listener, abort all its pending/active flows and cancel DNS/connect tasks. Any late completion, READY or flow for an invalidated rule MUST be discarded and its socket closed. A queued callback MUST recheck the exact rule instance before connecting, accepting data or delivering bytes. Bytes already delivered cannot be recalled. A Server policy revocation uses a reset with CHANNEL_REJECTED; ordinary local cancellation MAY use FIN. Cancelled streams are no longer parsed for late application messages. Other rules, terminal Channels and Sessions remain unaffected.

## 18.5 Flow establishment
```
forward-flow-open = [
  type: CHANNEL_OPEN,
  body: {
    0 => 5,
    1 => 0..4611686018427387903, ; rule stream ID
    ? 2 => forward-target,      ; required only for dynamic
    * int => any,
  },
]
forward-connected = [
  type: FORWARD_CONNECTED,      ; 0x4021; flow receiver to opener, once
  body: {
    0 => uint,                 ; status (§18.7)
    ? 1 => forward-socket,      ; required on success: target socket local endpoint
    * int => any,
  },
]
```
A listener accepts a TCP socket and opens one kind-5 Channel. For local/dynamic mode the Server connects the fixed/per-flow target. For remote mode the Client looks up the locally configured target for this exact rule and connects it; field 2 is forbidden, preventing Server-directed access to arbitrary Client services. No origin address supplied by the peer is trusted as authorization. Each flow requires an active rule on the same Connection, the correct opener direction, available limits and a fresh policy check. A well-formed flow with an unknown, cancelled, refused or wrong-mode rule is rejected with CHANNEL_REJECTED and causes no DNS lookup or target connection. Malformed fields remain protocol violations. The Client MUST reject unsolicited Server-opened flows without contacting a local target. A receiver MUST validate the rule and opener direction before initiating DNS, target connection or any other network side effect.

The receiver sends FORWARD_CONNECTED success only after the target TCP connection succeeds and it revalidates the rule. Field 1 reports getsockname of that target-side socket, a numeric address and actual port, not the destination's address and not the listener's address. On failure it omits field 1, sends FIN and accepts the opener's FIN; no application data is relayed. Before READY/CONNECTED, bounded local buffering is allowed but neither Endpoint may send FORWARD_DATA on the flow until success has been sent (receiver) or received (opener). A reset by the opener while connect is pending cancels that attempt. A timeout or cancellation after connect may mean a target observed a connection; it MUST NOT trigger an automatic retry.

Every newly accepted TCP connection requires a new Channel and independent target TCP connection. Target connections MUST NOT be pooled across flows. A flow has no terminal, Session, file permission or authority beyond its one target. A stale flow never falls back to another rule, Client, Attachment or Connection.

## 18.6 Data, half-close and failure
```
forward-data = [
  type: FORWARD_DATA,           ; 0x4022; either direction after success
  body: { 0 => bstr .size (1..61440), * int => any },
]
```
Each flow uses exactly one bidirectional QUIC stream with the existing length-prefixed CBOR framing (§7); bytes MUST NOT be sent unframed. Senders additionally respect the peer's maximum message size. Each direction's DATA byte strings concatenate into that direction's TCP byte stream with no transformation, replay or duplication. Empty data messages are forbidden. The receiver writes bytes in order, handling partial TCP writes; framing carries no message-boundary meaning for the target application. Backpressure MUST propagate between TCP and QUIC with bounded queues; one blocked flow MUST NOT prevent control processing or independent flow progress.

TCP read EOF means: finish queued DATA, then send QUIC FIN in that direction. On receiving FIN at a message boundary after success, deliver all preceding bytes before shutdown of the destination socket's write half. The reverse direction remains available until its own EOF. Both clean FIN directions and drained queues complete the flow and release its socket. After a complete CHANNEL_OPEN, FIN before CONNECTED or inside a subsequent frame is an incomplete exchange, never a successful empty flow; abort it locally with INTERNAL_ERROR. The base §8.3 rule for an initial stream ending before CHANNEL_OPEN still applies. A clean FIN after success is valid even with no DATA (empty TCP direction).

TCP reset, local I/O failure, stream reset/STOP_SENDING, timeout or rule revocation aborts the entire flow: close both halves of its local socket, discard undelivered queues and reset/stop the stream. Such failures MUST NOT be turned into graceful EOF or a successful delivery indication. This does not prove what a target application processed. No end-to-end application receipt, transfer hash, resume or exactly-once transaction guarantee is provided. A data/flow failure leaves the rule available for subsequent independent TCP connections unless that rule itself was revoked. Unexpected or duplicate messages on a live stream, data before success, invalid known fields and wrong message direction are Connection PROTOCOL_VIOLATION (§11.4).

## 18.7 Status, deadlines and bounds
FORWARD_READY and FORWARD_CONNECTED use their own status namespace. Unknown statuses are treated as a FAILED outcome, never as success; field 1 MUST be absent for every nonzero status:

| Status | Name | Meaning |
|---|---|---|
| 0 | OK | rule installed / target TCP connected |
| 1 | ACCESS_DENIED | policy or OS authority refuses |
| 2 | UNSUPPORTED | well-formed mode/address/facility unsupported |
| 3 | ADDRESS_IN_USE | listener address/port unavailable |
| 4 | NAME_NOT_FOUND | target DNS resolution failed |
| 5 | NETWORK_UNREACHABLE | no route to target network |
| 6 | HOST_UNREACHABLE | target host unreachable |
| 7 | CONNECTION_REFUSED | target TCP refusal |
| 8 | LIMIT_EXCEEDED | resource budget exhausted |
| 9 | FAILED | other local failure |

Statuses are not QUIC error codes. Unknown status values MUST be treated as FAILED with no success data used. A receiver MUST NOT disclose raw resolver errors, local paths or other secrets. Bind authorization and target policy refusal are ACCESS_DENIED; a target is not probed to refine that outcome. Establishment timeouts abort the affected Channel with TIMEOUT rather than inventing a success status. Policy refusal MUST NOT fall back to an unfiltered address or another mode.

RECOMMENDED deadlines: 10 s from rule CHANNEL_OPEN until READY; 10 s from flow CHANNEL_OPEN until CONNECTED, including DNS and all connect candidates; 10 s for a SOCKS greeting/request; 60 s without forwarding progress when application bytes are queued or a frame is partial. A successful idle flow and an idle rule have no service idle timeout by default. QUIC keepalive/Connection idle rules still apply (§13.3). Endpoints MUST bound opening, DNS, connects, stream-credit waits, partial-frame waits, queued TCP writes and teardown, and MUST enforce deadlines independently of blocked I/O. Receiver connect/setup timers start on complete CHANNEL_OPEN; an incomplete initial frame is separately bounded by 10 s from observing the stream. The local SOCKS total handshake deadline starts at TCP accept. The first applicable deadline wins; a trickle of bytes MUST NOT extend an absolute opening deadline. On opening timeout reset/stop only that Channel, and release provisional sockets; rule-opening failure also invalidates all its provisional flows. After both FINs, a blocked delivery queue remains subject to its progress deadline.

Both Endpoints MUST enforce the negotiated rule/flow maxima before allocating their expensive resources, and bound simultaneous accepts, queued sockets, DNS results, connect candidates, memory and tasks both per Connection and in aggregate. The Server MUST additionally limit by Account across Connections. RECOMMENDED aggregate Server defaults: 32 rules/256 flows per Account, 256 rules/2048 flows total, 16 DNS candidates per attempt, and at most 256 KiB of queued application bytes per flow across both directions. Implementations MUST also cap pending pre-Channel TCP/SOCKS sockets (RECOMMENDED 64 per Connection and 256 total). Client aggregate limits MUST cover all its Connections. Endpoints MUST also cap buffered bytes across flows (RECOMMENDED 32 MiB per Connection, plus 8 KiB per pending SOCKS handshake); reading pauses before bounds are exceeded, counting partial frames and outstanding native I/O. Opening, half-closed, cancelling and unreaped resources continue counting until actually released. A rejected admission closes the accepted socket promptly; it never queues an unbounded retry. Simultaneous admissions and revocation require atomic reservation/invalidation, including callbacks racing teardown. Lower configured aggregate limits may cause a well-formed opening to fail with LIMIT_EXCEEDED without changing the negotiated maxima. Server-opened streams and all pending waits are included, not exempt auxiliary resources.

## 18.8 Server and Client policy (PG004)
Forwarding MUST be disabled by default until explicitly enabled by trusted administrator policy (the operator-controlled Server configuration, including an unprivileged standalone Server). Authentication or permission to execute a terminal command alone MUST NOT enable it. Authorization is the intersection of global Server policy, Account policy and restrictions already present in the Authentication Context; key-policy provisioning remains PI002/RI003. The Server MUST enforce allowed modes, remote listener addresses/ports, ingress source restrictions when configured, permitted target addresses/ports for local and dynamic flows, and Account/global budgets. Listening and connecting MUST use no greater OS authority than authorized for the Account; a privileged Server MUST NOT make low-port binding or network-policy bypass available just because its service process can do so. If the platform cannot enforce this boundary it MUST refuse the operation.

For local/dynamic destinations, policy is checked before DNS using the requested name/port when name rules exist, and MUST be checked against each numeric candidate/port before any connect attempt. Name authorization alone MUST NOT bypass numeric destination restrictions. Resolution MUST NOT use search suffixes. The connector MUST use the already checked numeric address directly without a second name lookup. Re-resolution on another flow requires another check; DNS rebinding, alternate A/AAAA results and IPv4-mapped addresses cannot broaden the allowed set. Disallowed candidates are never probed; if none are allowed return ACCESS_DENIED. An empty/failed DNS resolution returns NAME_NOT_FOUND. Limits/deadlines apply even to refused or attacker-controlled DNS results. Local and private addresses, including loopback, MAY be explicitly allowed for intended forwarding; they are not implicitly granted. IPv6 scope-dependent destinations are refused rather than guessed.

Remote binds MUST match policy before creating listeners; wildcard/non-loopback exposure needs explicit administrator permission as well as Client configuration. Port 0 requires permission for the actual assigned port: bind provisionally, validate getsockname against policy, and close without accepting if disallowed. An ingress filter MUST run before dispatching the flow. A remote rule intentionally delegates reachability of its Server listener to one Client-configured target; the Server cannot verify that target and MUST NOT claim destination filtering there. Administrators that cannot accept this delegation must disable remote mode. The Client MUST apply its own destination policy, exact local rule binding and checked numeric DNS/connect procedure on every remote flow. No peer message may replace that target. Compromise of an explicitly configured Client target is outside destination-filter guarantees.

Policy changes denying an existing rule or flow MUST invalidate it and abort affected sockets on local observation, without awaiting peer acknowledgement. Where only one flow's destination is revoked, unrelated allowed flows may remain. Late asynchronous completions MUST NOT reopen access. These controls are enforced by the Endpoint owning the resource; a malicious peer cannot be trusted to close its own resources. Policy diagnostics SHOULD remain local. Forwarding exposes services reachable by the connecting Endpoint, including services without their own authentication; trusted local configuration MUST identify the direction, bind exposure and target. The tunnel encrypts only the Client–Server segment, not the external TCP legs.

## 18.9 Dynamic forwarding (PG003)
The Client implements a deliberately restricted SOCKS5 CONNECT proxy using the packet layouts and reply meanings of [RFC 1928](https://www.rfc-editor.org/rfc/rfc1928.html), §§3–6. This is not a claim of complete RFC 1928 conformance: GSSAPI, username/password, BIND and UDP ASSOCIATE are not part of profile 1. It accepts only version 5, method 0 (no authentication), command 1 (CONNECT), reserved byte 0, and IPv4/domain/IPv6 address forms. No acceptable method yields method 0xFF and close; BIND/UDP yield REP 0x07; unsupported address forms yield REP 0x08. Malformed/incomplete requests are refused and closed under the handshake deadline. Implementations MUST bound greeting, request and early-data buffering; extra pipelined bytes cannot be forwarded before CONNECTED success.

SOCKS has no local user authentication in this profile. The listener MUST default to loopback. Exposing it on a non-loopback address requires explicit local configuration acknowledging that reachable users can consume this Account's forwarding authority, plus a configured local ingress allowlist. An unavailable ingress enforcement facility means refusal. SOCKS is terminated at the Client; greeting/authentication/request bytes are never sent as target application data. One SOCKS CONNECT creates one kind-5 flow under a dynamic rule. Numeric forms are normalized per §18.3; domain form is validated/converted to that ASCII representation locally, then sent as a name to the Server for resolution. A request naming an unsupported scoped destination is refused. Applications that resolve names themselves can still perform local DNS; the proxy MUST NOT claim to prevent those leaks.

Only after FORWARD_CONNECTED success does the Client send SOCKS REP 0x00, using its field 1 as BND.ADDR/BND.PORT, and begin data relay. Failure maps ACCESS_DENIED to 0x02, NETWORK_UNREACHABLE to 0x03, NAME_NOT_FOUND/HOST_UNREACHABLE to 0x04, CONNECTION_REFUSED to 0x05, UNSUPPORTED address to 0x08 and other failures (including TIMEOUT/limits/cancellation) to 0x01. Failure replies use an all-zero IPv4 bound address/port and close promptly (within the RFC's 10 s maximum). A locally failed reply write aborts the flow. After successful reply, TCP EOF/reset and QUIC FIN/reset follow §18.6. No SOCKS request is automatically retried on Connection loss. UDP remains outside PG001–PG004 even though the tunnel itself runs over QUIC/UDP.


# Security Considerations
Collected from all sections; consolidated in Tier PJ.
- TCP forwarding requires explicit local configuration and administrator/Account policy, checks every numeric DNS candidate, and never accepts a Server-selected Client target. Rules end with their Connection, cancellation aborts pending work and live sockets, and only the STPT segment is encrypted (§18).
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
- No key probing, uniform failures and checks completed regardless of the failing one prevent enumeration of Accounts and authorized keys, also by timing (§12.2, §12.4).
- No passwords: nothing a phishing or compromised Server could capture and reuse (§12.1).
- Before authentication errors carry no reason text; exact reasons stay in the Server log (§13.1).
- A QUIC or TLS implementation may add its own text to a transport close after a failed handshake; the text can identify the implementation, against the intent of §10.3. It is suppressed where the implementation allows; a receiver never acts on it (§13.1).
- Deadlines, limits and QUIC Retry bound the resources an unauthenticated peer can consume (§13.3–§13.5).
- Only the Client opens terminal Channels; the Server cannot start an interactive context on the Client (§14.3).
- Every terminal Channel is an authorized Attachment; Session termination is a separate explicit request (§14.3).
- Environment variables pass only through a Server allowlist, closing attacks such as `LD_PRELOAD`, `BASH_ENV` or Shellshock (CVE-2014-6271); the Client does not leak its environment (§14.6).
- The argument-vector command form avoids shell quoting injection; both forms run through the Account's shell, so a disabled shell (`nologin`) also refuses commands; the terminal type is restricted to a safe character set (§14.6).
- Terminal output reaches the Client's terminal emulator unfiltered; hostile escape sequences (e.g. clipboard writes by OSC 52, title or status reports injected back as input) are a risk of the emulator. Client-side protection is an open question (DECISIONS.md §7) (§14.7).
- Output is never buffered without bound: QUIC flow control stops reading at the source (§14.7).
- A small resize causes a large redraw; coalescing and Server rate limiting bound this amplification (§14.8).
- Explicit signals carry the Account's permissions, never those of the Server process, so they cannot reach processes the user could not signal locally (e.g. root processes started with `sudo`) (§14.9).
- A bounded grace period after the primary process exits keeps background processes from holding a Channel open indefinitely (§14.10).
- Random Session IDs are never reused, even across Server restarts, so input cannot go to a different Session that got the same ID (§15.1).
- Requests have their own results and never close the Connection on failure (§8.4).
- Session termination escalates to KILL after a bounded grace period and signals with the Account's permissions only (§15.6).
- A limit of Sessions per Account bounds the resources an authenticated user can hold on the Server (§15.8).
- A stalled Client is detached after a bounded time, so it cannot block the processes of a Session or the other Attachments (§15.9).
- Read-only Attachments prevent accidental input but are not a security boundary within an Account (§15.9).
- The Session list shows only the Sessions of the Account; Attachment addresses and commands are visible to every key of that Account, which matters for Accounts shared by several people (§15.10).
- The output buffer keeps recent output, including secrets shown on screen, in Server memory until the Session ends and replays it to any Attachment of the Account; its size is bounded (§15.11).
- Replay needs no terminal emulator on the Server: no parser of hostile output, no extra attack surface (§15.11).
- Reattach after a lost Connection requires full user authentication; there is no resume token to steal (§15.12).
- A Session ID grants no access; an ID of another Account's Session is answered exactly as an unknown one, so Sessions of other Accounts cannot be probed (§15.2).

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
- draft-09 (2026-10-11): errors, timeouts and resource limits (§13); references to P items replaced by section numbers.
- draft-10 (2026-10-13): terminal Channel: channel kinds, opening, single stream (§14).
- draft-11 (2026-10-13): terminal parameters: mode, command, PTY, environment, attach (§14.6).
- draft-12 (2026-10-13): terminal I/O: data, stderr, EOF, backpressure (§14.7).
- draft-13 (2026-10-13): terminal resize (§14.8).
- draft-14 (2026-10-13): signals (§14.9).
- draft-15 (2026-10-13): exit status and Channel close (§14.10).
- draft-16 (2026-10-13): Session ID, ownership, Session name (§15); error codes SESSION_NOT_FOUND, SESSION_NAME_IN_USE (§13.2).
- draft-17 (2026-10-13): authentication checks completed before one decision (§12.4); rejection reasons in the Server log (§13.1).
- draft-18 (2026-10-13): control requests and REQUEST_RESULT (§8.4); Session creation, detach, termination, exit and removal, Retention Policy (§15.5–§15.8); message types SESSION_TERMINATE, REQUEST_RESULT.
- draft-19 (2026-10-13): detach timeout requested by the Client (§14.6, §15.8).
- draft-20 (2026-10-13): attach authorization, Attachment ID, several Attachments, attach modes shared and takeover, read-only, terminal size, stalled Attachments, detach request (§15.9); message type SESSION_DETACH; error code ATTACHMENT_NOT_FOUND.
- draft-21 (2026-10-13): Session list with Attachments, paging (§15.10); message type SESSION_LIST.
- draft-22 (2026-10-13): output buffer, output offsets, replay and resume, redraw on attach (§15.11); connection migration required, reattach after Connection loss (§6.1, §15.12).
- draft-23 (2026-10-17): key exchange group order of the Client, Server handling of a Client preferring X25519 (§6.2).
- draft-24 (2026-10-17): reason phrase rule limited to closes initiated by the STPT implementation; text added by the QUIC or TLS implementation, diagnostics only (§13.1).
- draft-25 (2026-10-17): GREASE values are exactly 16 per kind, with one repeated digit (§11.5); Client deadline for SERVER_HELLO (§13.3).
- draft-26 (2026-10-17): Client deadline for AUTH_RESULT (§13.3).
- draft-27 (2026-10-17): streams that end or are reset before CHANNEL_OPEN (§8.3).
- draft-28 (2026-10-18): the argument vector also runs through the Account's shell, quoted (§14.6).
- draft-29 (2026-10-18): a fresh replay that starts at offset 0 is replayed whole (§15.11).
- draft-30 (2026-10-18): terminal modes followed by the Server and sent on attach (§14.3, §15.11); redraw by a brief size change.
- draft-31 (2026-10-18): a fresh replay starts with the current screen (§15.11); replaces the offset 0 rule of draft-29.
- draft-32 (2026-11-03): Client side of resume: terminal modes on a resume without loss (§15.11); key checks and input during automatic reattach (§15.12).
- draft-33 (2026-11-03): a Client notices a change of its address and migrates at once (§15.12).
- draft-34 (2026-11-08): proposed file transfer Channel envelope (§16). Corrected the stale specification header, which still named draft-31 despite the draft-32 and draft-33 history entries.
- draft-35 (2026-11-08): PE001 Channel model decided: independent of Sessions, one stream per Channel, sequential operations and negotiated parallel Channel limit; registered capability 1, Channel kind 2 and FILES_READY (0x4000).
- draft-36 (2026-11-17): PE002 file operation model: sequential requests and final results, operation statuses, bounded listing, Account authorization, atomic upload publication and explicit replacement. Transfer framing, resume and path/metadata encodings remain PE003–PE005.
- draft-37 (2026-11-17): PE003 sequential bounded file blocks, declared lengths, incremental SHA-256, explicit end/receipt/result synchronization, large-file limits and Channel-local interruption. Resume and path/metadata encodings remain PE004–PE005.
- draft-38 (2026-11-17): PE003 makes BLAKE3 the required and default file hash and adds optional SHA-256. Each transfer confirms its algorithm before file data and retains that choice for resume.
- draft-39 (2026-11-17): PE004 adds optional read and write resume with verified prefixes, Account-bound upload tokens, bounded staging retention and publication safeguards. Path and metadata encodings remain PE005.
- draft-40 (2026-11-17): PE005 specifies absolute POSIX and Windows paths, listing and stat metadata, exact path binding for resume, and safe destination metadata. The draft definition of file transfer profile 1 now covers PE001–PE005.
- draft-41 (2026-11-21): scenario review of Tier PE clarifies required rename fields, empty and special directory entries, integrity-error precedence, operation-wide progress deadlines and retained upload parent identity; synchronizes the timeout and resource-limit registries. Review results: docs/PE_REVIEW.md. No wire allocations or program implementation changed.
- draft-42 (2026-11-21): records the RE001 Channel envelope implementation status. File operations remain unimplemented; no normative wire changes.
- draft-43 (2026-11-21): records RE002 POSIX file operations and verified transfer implementation, with explicit remaining limits. No normative wire changes.

- draft-44 (2026-11-22): records RE003 explicit verified upload/download resume, bounded Server-instance retention and Client checkpoints. No normative wire changes.

- draft-45 (2026-11-22): records RE004 path/metadata implementation on available platforms, with native Windows Server behavior still gated by G2/RC008 and Tier RE review pending. No normative wire changes.

- draft-46 (2026-11-22): records RC008 ConPTY/current-Account Windows terminal Server, retired G2 and separate Windows file-service gate G3. No normative wire changes.

- draft-47 (2026-11-22): records native Windows current-Account file service, handle-relative volume/reparse enforcement and private atomic upload publication; resolves G3. No normative wire changes.

- draft-48 (2026-11-22): records the completed Windows/Linux Tier RE scenario review and implementation fixes for active FIN/deadlines, verified Client publication and private checkpoint files. No normative wire changes.

- draft-49 (2026-11-22): records the Windows-only 0.4.1-2 follow-up that creates Client resume state with atomic private descriptors and explicit owner; distinguishes final Windows validation from unchanged Linux 0.4.1-1 evidence. No normative wire changes.
- draft-50 (2026-11-24): defines Client consent, grant scope, enforcement and revocation for PF001 (§17). No wire allocations.
- draft-51 (2026-11-24): defines Session-local request origin, deterministic Attachment selection and delivery to one Client (§17.4, PF002).
- draft-52 (2026-11-24): assigns capability 2 and kind 3, with one-shot Client consent, verified read/create, refusal, cancellation and deadlines (§17.5–§17.7).
- draft-53 (2026-11-24): adds the explicitly approved read-only directory mount extension, scoped operations and independent teardown (§17.8–§17.10, PF004).
- draft-54 (2026-11-24): Tier PF scenario review clarifies target pinning, consent identity, cancellation and mount handoff/revocation (§17). Wire allocations are unchanged.
- draft-55 (2026-11-24): records RF001 Client permission state implementation. No wire changes.
- draft-58 (2026-11-24): records the Linux implementation of PF004 in 0.4.5-0. No wire changes.
- draft-57 (2026-11-24): records the RF003 one-shot implementation and trusted local terminal in 0.4.4-0. Wire assignments are unchanged.
- draft-56 (2026-11-24): records the RF002 Session-local command and target selection in 0.4.3-0. No wire changes.

- draft-59 (2026-11-25): specifies TCP local/remote/dynamic forwarding, Connection-scoped rules, capability 4 and kinds 4/5, framed flows with half-close, SOCKS5 CONNECT, policy, cancellation, deadlines and resource limits (§18; PG001–PG004). Unimplemented; scenario acceptance criteria are in docs/PG_REVIEW.md.

- draft-60 (2026-11-25): records program 0.5.0-0 implementation of PG001–PG004 (RG001–RG004), with Connection-scoped TCP forwarding, restricted SOCKS5 CONNECT, Account-authorized native sockets and explicit opt-in policy. No wire allocations or normative requirements changed.

- draft-61 (2026-11-25): records the full RG scenario review and program 0.5.0-1 fixes for early-flow cancellation, retained Account quota identity, TIMEOUT signalling, blocked replies and foreground route monitoring. Wire allocations and normative requirements are unchanged.
