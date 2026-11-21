# Tier PE scenario review

Date: 2026-11-21. Specification reviewed: draft-40, with clarifications incorporated in draft-41. Scope: PE001–PE005 and their interaction with framing, negotiation, errors and resource limits.

This is a specification walkthrough, not runtime testing, a formal proof, or a complete CDDL validator run. At review time, file transfer was not implemented in `stpt` and RE001–RE004 were open. Subsequent RE001/RE002 implementation and runtime validation are recorded in DECISIONS.md §19–§20. The scenario table below remains a record of the earlier specification review. Existing PE completion dates denote specification work, not this review or implementation.

## Scenario results

| Scenario | Required outcome and specification coverage |
|---|---|
| Capability absent or rejected | Terminal and Session services remain available; no file Channel may be used (§11.4, §16.2). |
| Channel rejected or opening stalls | Stream-local rejection or opening timeout; other Channels remain usable (§16.2). |
| Parallel Channels exceed the selected limit | LIMIT_EXCEEDED for the excess opening, with no unlimited stream or resource reservation (§16.3). |
| Invalid fields, direction, phase or operation ID | PROTOCOL_VIOLATION; an unknown well-formed operation receives UNSUPPORTED without filesystem effects. Rename requires a destination (§16.5). |
| Empty file | No FILE_DATA; FILE_END carries count zero and the empty-sequence digest. Upload still publishes and returns a final result; download still exchanges receipt and result (§16.10–§16.12). |
| Ordinary file | Accepted length and algorithm, contiguous data, end verification and final result. FILE_ACCEPTED and FIN are never success (§16.5, §16.10–§16.12). |
| File above 4 GiB | Same messages and bounded memory; checked integer arithmetic. The common bound is 2^61 - 1 bytes; smaller local limits may reject the request (§16.10). |
| Largest data chunk | A 61440-byte chunk with maximum uint64 operation ID and maximum valid offset needs 61469 payload bytes without extension fields, below the 65535-byte minimum limit. The 4-byte QUIC length prefix is separate from that payload limit (§7.2, §16.10). |
| Empty directory or native dot entries | Empty listing has only a final OK result; native `.` and `..` entries are omitted. Enumeration stays bounded (§16.6). |
| Listing fails after some batches | Earlier entries form an incomplete listing. An unrepresentable name fails with UNSUPPORTED; an entry too large fails with LIMIT_EXCEEDED (§16.6, §16.19). |
| Account access denied or revoked | Check each actual access under Account authority, including staging, cleanup and publication. No privileged bypass; policy can conceal existence (§16.7–§16.9, §16.16). |
| Disk/quota exhaustion | Before acceptance, a file failure result; during upload reception, abort the Channel rather than drain arbitrary data or send a premature result. Never publish partial data (§16.8, §16.12–§16.13). |
| Source EOF or growth | Early EOF fails; appended bytes are excluded. In-place changes need not be detected, and a successful hash does not imply a snapshot (§16.11). |
| Wrong count or digest | INTEGRITY_ERROR after a complete end boundary; no upload publication. Count mismatch takes precedence even when FILE_END reports a sender failure (§16.11–§16.12). |
| Empty chunk, gap, overlap or data beyond length | PROTOCOL_VIOLATION, distinct from an end-marker integrity mismatch (§16.10). |
| Failure within a frame | Complete the frame before any following message, or abort the Channel. FIN inside a frame remains a framing violation under §7.2; a reset is the cancellation/failure mechanism (§16.4, §16.12). |
| Upload interrupted before complete FILE_END | No publication. Retain only an opted-in eligible checkpoint, otherwise clean staging. Retention is bounded and never exposes the partial destination (§16.8, §16.16). |
| Complete FILE_END received, result lost | Consume the upload token before publication may begin. Outcome can be unknown; no automatic resume or fresh modifying retry. Automatic resume is allowed only when the Client can establish that no complete FILE_END was sent; refusal never implies a fresh transfer (§16.16–§16.17). |
| Download verification/local writes fail | No OK receipt. Report the failure after consuming the end boundary, or abort without draining an unbounded remainder (§16.11–§16.12). |
| Verified download, final result lost | Local verified data may exist, but protocol success cannot be reported (§16.8, §16.12). |
| Download resume length/prefix changed | RESUME_UNAVAILABLE before acceptance. A matching prefix permits the current suffix, not restoration of an immutable historical version (§16.14–§16.15). |
| Upload resume local prefix changed | Client compares the Server's digest with its own source prefix and cancels on mismatch, without sending suffix data (§16.16). |
| Resume at total length | No FILE_DATA; full-file digest and ordinary completion exchange are still required (§16.14). |
| Busy, expired, foreign or consumed token | BUSY only for a matching active upload; bounded retry is allowed. Other unavailable state receives RESUME_UNAVAILABLE without disclosure to another Account or implicit fresh transfer (§16.17). |
| Destination parent replaced before resume | Store the original directory identity and reject resume if the exact saved path now resolves to another parent. Publication targets the verified directory under current authorization (§16.16, §16.18). |
| Server restart | Old upload tokens are rejected; abandoned staging is safely cleaned without publication (§16.16). |
| Symlinks and concurrent path replacement | POSIX final-entry semantics vary explicitly by operation; all actual accesses must remain race-safe and under Account authority (§16.6, §16.9). |
| Windows reparse point or namespace alias | No reparse traversal; only specified final-entry inspection/mutation is allowed. Unsupported semantics are refused, not approximated (§16.18). |
| Non-UTF-8 POSIX name or cross-platform collision | Preserve POSIX bytes. Validate remote names before local use, reject lossy conversions and collisions or require explicit distinct names; escape controls for display (§16.18–§16.19). |
| Slow peer or lengthy prefix verification | Bound queues, descriptors, staging and work. A configured operation progress timeout aborts only the Channel; a peer may time out during remote local work, including prefix hashing (§16.9, §16.13, §16.16). |

## Clarifications made

- Required rename destination and permitted replacement field are explicit.
- Empty listings and native dot-entry omission are explicit.
- Count/integrity failure has consistent precedence in upload results, download receipts and final download results.
- The timeout and resource-limit registries include file operations. Progress deadlines cover setup and resume verification as well as bulk data, with the remote-work visibility limitation stated.
- Retained upload state explicitly records the original destination parent identity, as already required by the resume path check.

No unresolved protocol-policy choice was identified in these scenarios. The draft is ready to guide RE implementation; filesystem API feasibility, concurrency races, cancellation, quotas, malformed-peer handling and interoperation still require implementation tests. No claim of production security or runtime conformance is made.

## Mechanical checks

Passed: review section references, consecutive draft history through draft-41, matching specification header, paired code fences, local Markdown link targets, PE completion and RE open statuses, and the program architecture's draft reference. An independent preferred-CBOR encoding confirmed the maximum FILE_DATA payload size above; integer encodings were checked around 4 GiB and at the common maximum. The SHA-256 input bound is `(2^61 - 1) * 8 = 2^64 - 8` bits. These checks do not validate all CDDL schemas or replace implementation tests. `git diff --check` passed in both repositories.

## Platform references

Windows naming and reparse-point assumptions were checked against Microsoft's [Naming Files, Paths, and Namespaces](https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file) and [Reparse Points](https://learn.microsoft.com/en-us/windows/win32/fileio/reparse-points). These sources describe the platform; the deliberately narrower profile rules remain normative in PROTOCOL.md §16.18.
