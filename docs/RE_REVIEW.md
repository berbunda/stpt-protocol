# Tier RE implementation review

Date: 2026-11-22. Program 0.4.1-1; specification draft-48. Scope: RE001–RE004 and all 30 scenarios of [PE_REVIEW.md](PE_REVIEW.md).

The scenario review is complete on native Windows and Linux, with fixes committed alongside the results. The [program's detailed scenario report](https://github.com/berbunda/stpt/blob/main/docs/RE_REVIEW.md) maps each scenario to shared or native evidence, records reproductions, commands and limitations. The PE review remains a historical specification walkthrough.

## Findings resolved

- Active FIN is a Channel interruption, not a malformed file message or a reason to lose other Channels. Deadline/cancellation monitoring remains installed after input completion.
- Client publication uses the verified staging object through pinned native handles/directories. Substituting a temporary source pathname cannot turn a verified download into different published bytes.
- Client resume state checks the owner and private access policy on the actual opened files before writing. Windows protection includes freshly created files and refuses unrelated owners/reparse entries; Unix rejects unsafe modes or owners.

These correct implementation behavior under §16.4, §16.8–§16.9 and §16.14–§16.20. No normative wire message, version, hash, status or capability definition changes.

## Validation

| Check | Result |
|---|---|
| Windows standard suite | 273 passed; manual large case excluded by default |
| Windows explicit large transfer | 1 passed: 4 GiB + 64 KiB, full digest and completion exchange |
| Linux standard suite | 308 passed; 6 historical unrelated skips plus the separately run large case |
| Linux explicit large transfer | 1 passed: same logical length, sparse source and bounded hash sink |
| Root Linux file suite | 61 passed; actual Account separation and foreign-token refusal included |
| Clippy all targets, warnings denied | Passed on Windows and Linux |
| Format/diff/version/roadmap/document checks | Passed |

Shared grammar/phase/resource tests and native filesystem/QUIC cases cover every row of PE_REVIEW.md. Disk-full is checked through native status mapping and safe error injection, not by filling the user's disks. A 4-GiB staged upload and a new direct cross-OS network run are outside the execution evidence. Full details are in the program report; no production-security or universal-filesystem conformance claim is made.

G2 and G3 remain resolved. The supported Windows Account/token restriction, safe UNSUPPORTED refusal for existing-target uploads, unavailable native facilities, informational-only metadata and potentially unknown lost modifying outcomes remain explicit implementation limits. The review introduces no later Tier feature, new dependency or policy change.
