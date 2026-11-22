# stpt-protocol
Secure Terminal Processing Tools is realisation of remote CLI control via QUIC transport protocol

Specification: `specs/PROTOCOL.md`. Roadmap of the protocol: `docs/ROADMAP.md`.

PE001–PE005 define draft file transfer profile 1; the program implements the Channel envelope and ordinary POSIX file operations/transfers, explicit verified resume, Windows drive encoding/Client parsing and informational metadata (RE001–RE004 on available platforms). Windows terminal Server is implemented with ConPTY for its current Account/token (RC008); native Windows file service remains unavailable under G3; Tier RE scenario review is pending. Implementation limits are recorded in specs/DECISIONS.md §20–§23. The [Tier PE scenario review](docs/PE_REVIEW.md) records specification checks and remaining implementation validation.
