# stpt-protocol
Secure Terminal Processing Tools is realisation of remote CLI control via QUIC transport protocol

Specification: `specs/PROTOCOL.md`. Roadmap of the protocol: `docs/ROADMAP.md`.

PE001–PE005 define draft file transfer profile 1; the program implements the Channel envelope and ordinary POSIX file operations/transfers and explicit verified resume (RE001–RE003). Windows Server paths and extended metadata remain RE004; implementation limits are recorded in specs/DECISIONS.md §20–§21. The [Tier PE scenario review](docs/PE_REVIEW.md) records specification checks and remaining implementation validation.
