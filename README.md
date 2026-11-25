# stpt-protocol
Secure Terminal Processing Tools is realisation of remote CLI control via QUIC transport protocol

Specification: `specs/PROTOCOL.md`. Roadmap of the protocol: `docs/ROADMAP.md`.

PE001–PE005 define draft file transfer profile 1; the program implements the Channel envelope and ordinary POSIX file operations/transfers, explicit verified resume, Windows drive encoding/Client parsing and informational metadata (RE001–RE004 on available platforms). Windows terminal Server is implemented with ConPTY for its current Account/token (RC008); native Windows local-volume file service now uses that Account/token and resolves G3 (program 0.4.1-0); the [Tier RE scenario review](docs/RE_REVIEW.md) is complete in program 0.4.1-2, with fixes and real transfers above 4 GiB on both platforms. Implementation limits are recorded in specs/DECISIONS.md §20–§26. The [Tier PE scenario review](docs/PE_REVIEW.md) records the earlier specification checks.

[Protocol §17](specs/PROTOCOL.md) defines Client file permissions (PF001), routing to one Attachment (PF002), the Server-opened Channel for one locally approved read or create operation (PF003), and explicitly approved read-only directory mounts tied to that Attachment (PF004). These services are specified in draft form.

The [Tier PF scenario review](docs/PF_REVIEW.md) records the draft-54 routing, cancellation and mount-revocation scenarios for RF implementation.

The [combined RF review](docs/PF_REVIEW.md#combined-rf-implementation-review-045-1) records the 0.4.5-1 implementation checks and the fix for revoking a mount during a blocked read.

Program 0.4.2-0 implements the Client permission state for PF001. At that milestone, the Session-local request path and confirmation interface were still pending.

Program 0.4.3-0 implements the Session-local command and Attachment selection for PF002. Program 0.4.4-0 implements the PF003 Server-opened Channel for read and create, with one-time approval in a separate local terminal. Program 0.4.5-0 adds read-only Client directory mounts on Linux, with local revocation and automatic teardown when the bound Attachment, Session or Connection ends.

[Protocol §18](specs/PROTOCOL.md) defines draft TCP local, remote and dynamic (restricted SOCKS5 CONNECT) forwarding, Connection-scoped lifetime and Server/Client policy (PG001–PG004, draft-59). The [Tier PG scenario review](docs/PG_REVIEW.md) records acceptance criteria. Program 0.5.0-0 implements RG001–RG004 (recorded in draft-60): the foreground `forward` command, opt-in per-Account policy and native Account-authorized sockets. The CLI runs one rule per Connection. UDP forwarding stays in the on-request backlog.

Program 0.5.0-1 completed the [full 41-scenario RG review](docs/RG_REVIEW.md), with cancellation/quota/deadline fixes, foreground migration monitoring and Windows/Linux validation. Specification draft-61 records the result without changing the forwarding wire profile.
