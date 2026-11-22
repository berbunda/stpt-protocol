# stpt-protocol
Secure Terminal Processing Tools is realisation of remote CLI control via QUIC transport protocol

Specification: `specs/PROTOCOL.md`. Roadmap of the protocol: `docs/ROADMAP.md`.

PE001–PE005 define draft file transfer profile 1; the program implements the Channel envelope and ordinary POSIX file operations/transfers, explicit verified resume, Windows drive encoding/Client parsing and informational metadata (RE001–RE004 on available platforms). Windows terminal Server is implemented with ConPTY for its current Account/token (RC008); native Windows local-volume file service now uses that Account/token and resolves G3 (program 0.4.1-0); the [Tier RE scenario review](docs/RE_REVIEW.md) is complete in program 0.4.1-2, with fixes and real transfers above 4 GiB on both platforms. Implementation limits are recorded in specs/DECISIONS.md §20–§26. The [Tier PE scenario review](docs/PE_REVIEW.md) records the earlier specification checks.
