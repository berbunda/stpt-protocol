# STPT Gates

Restrictions affecting feature availability (RULES.md §1.4.3). Identifiers are permanent and MUST NOT be reused.

- G1  Server-side ECH: `rustls` supports ECH on the client only; server-side support is on the rustls roadmap (rustls/rustls#1980). ➕ 2026-10-10
- G2  Resolved ✅ 2026-11-22 (RC008, program 0.4.0-0). Historical restriction — Server mode on Windows: the Server needs a Unix PTY; on Windows `stpt server` is refused until ConPTY support (RC008). ➕ 2026-10-14 Windows terminal Server now uses ConPTY and the current process Account/token. Windows filesystem service is separately gated by G3.

- G3  Windows Server file service: no native Account/volume/reparse/staging backend yet. The Windows Server selects no file capability, so no Windows file Channel is offered or accepted. Client Windows path/metadata support remains available. ➕ 2026-11-22
