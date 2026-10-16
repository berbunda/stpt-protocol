# STPT Gates

Restrictions affecting feature availability (RULES.md §1.4.3). Identifiers are permanent and MUST NOT be reused.

- G1  Server-side ECH: `rustls` supports ECH on the client only; server-side support is on the rustls roadmap (rustls/rustls#1980). ➕ 2026-10-10
- G2  Server mode on Windows: the Server needs a Unix PTY; on Windows `stpt server` is refused until ConPTY support (RC008). ➕ 2026-10-14
