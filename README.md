# stpt-protocol
Secure Terminal Processing Tools is realisation of remote CLI control via QUIC transport protocol

Specification: `specs/PROTOCOL.md`. Roadmap of the protocol: `docs/ROADMAP.md`.

PE001–PE005 define draft file transfer profile 1; the program implements the Channel envelope and ordinary POSIX file operations/transfers (RE001–RE002). Resume, Windows Server paths and extended metadata remain RE003–RE004; implementation limits are recorded in specs/DECISIONS.md §20. The [Tier PE scenario review](docs/PE_REVIEW.md) records specification checks and remaining implementation validation.
