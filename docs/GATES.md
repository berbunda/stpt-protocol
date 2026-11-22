# STPT Gates

Restrictions affecting feature availability (RULES.md §1.4.3). Identifiers are permanent and MUST NOT be reused.

- G1  Server-side ECH: `rustls` supports ECH on the client only; server-side support is on the rustls roadmap (rustls/rustls#1980). ➕ 2026-10-10
- G2  Resolved ✅ 2026-11-22 (RC008, program 0.4.0-0). Historical restriction — Server mode on Windows: the Server needs a Unix PTY; on Windows `stpt server` is refused until ConPTY support (RC008). ➕ 2026-10-14 Windows terminal Server now uses ConPTY and the current process Account/token. Windows filesystem service is recorded separately under G3.

- G3  Resolved ✅ 2026-11-22 (program 0.4.1-0). Historical restriction — Windows Server file service lacked native Account/volume/reparse/private-staging support and selected no file capability. ➕ 2026-11-22 Native local-volume file operations, verified transfers/resume and metadata now use the current Account/token, pinned directory handles and private atomic publication. Existing-target upload replacement remains unsupported on both primary platforms; the complete Tier RE scenario review passed with fixes in 0.4.1-1 (see [RE_REVIEW.md](RE_REVIEW.md)).
