# Tier PF scenario review

2026-11-24. This walkthrough followed the Client, Server and Session-local caller through PF001–PF004 in draft-53 and checked the outcomes below against the revised [specification](../specs/PROTOCOL.md). Draft-54 incorporates the corrections. The scenarios are acceptance criteria for RF; no RF implementation was exercised.

| Starting state and event order | Required outcome | § |
|---|---|---|
| A local caller supplies another Session's ID or only knows STPT_SESSION | Refuse without contacting any Client. Establish origin through the protected Session context. | 17.4 |
| No Attachment, or the requested ID has detached | Fail locally. A later join receives no queued proposal. | 17.4 |
| A and B share the Session; only B offers file access, or A is terminal-read-only | With no target, report ambiguity. With explicit A, evaluate A alone and never fall back to B. | 17.4 |
| A is alone at selection; B joins before dispatch | Pin A at selection. B's arrival neither reroutes nor cancels. If A leaves instead, fail even if B is now alone. | 17.4 |
| Same numeric Attachment ID in a different Session, or A reconnects on a new Connection | Match the entire binding. Neither the other Session nor the new Attachment receives the old prompt. | 17.1–17.4 |
| Capability 1 only; capability 2 only; capabilities 2 and 3 together | No reverse Channel in the first case. Action 3 is rejected without prompting in the second and eligible for mount approval only in the third. | 11.4, 17.5, 17.8 |
| One live mount uses the only slot; another caller requests a file | Enforce the slot limit through teardown. No second prompt over the limit; unrelated terminal/control traffic continues. | 17.5, 17.8 |
| Wrong-direction frame, unsolicited START, or duplicate approval on a live exchange | Connection protocol violation. A well-formed refusal or a stale target is instead a Channel-local outcome. | 17.6–17.7 |
| Forged terminal prompt, no trusted UI, dismissal or consent timeout | No grant or local path disclosure. Before admission reject the Channel; after PENDING send DENIED for dismissal, abort with TIMEOUT on expiry. | 17.1, 17.6–17.7 |
| Local file, destination parent or directory root is replaced between confirmation and START | Use the approved object or fail. Reopening the replacement based solely on its name cannot satisfy consent. | 17.2, 17.6, 17.8 |
| Caller cancels before START while APPROVED is in flight | Cancellation wins at the Server. Ignore late approval and send no START. No operation can have started. | 17.6–17.7 |
| START already sent; cancellation reaches Client before admission or publication | Invalidate locally, discard staging and drop late work. The Server reports unknown if it cannot rule out publication. | 17.3, 17.7 |
| Create publishes before cancellation; success result crosses cancellation or is lost | Leave the committed file in place. A previously delivered success stands; otherwise a caller lacking a definitive result can receive unknown. No automatic retry. | 17.6–17.7 |
| Empty create, wrong digest, destination created by another process | Empty END still needs its digest. Integrity failure leaves no published file; a destination race preserves the existing entry. | 17.6 |
| Sender sends END and FIN; result is delayed, or FIN arrives without END | In the first case continue waiting under deadline; in the second fail the incomplete exchange. FIN alone never proves success. | 17.6–17.7 |
| A approves a mount while B shares the Session | Warn about Server root and shared Session access before explicit local yes. Processes driven by B can use the mounted view until A's grant ends. B cannot renew A's grant. | 17.1, 17.8 |
| Caller abandons setup or A detaches while MOUNT_READY is in flight | If invalidation wins before handoff, discard READY and dismantle provisional state. If installation wins, caller exit is harmless but A's detach tears down the view. | 17.7–17.8 |
| B detaches, A detaches through takeover, or the Session exits | B's unrelated detach preserves the view. Either of the latter events ends A's grant without waiting for peer acknowledgement. | 17.3, 17.10 |
| Link or reparse traversal, nested mount, or an opened descendant moves outside the approved tree | Refuse traversal. A held handle alone cannot authorize subsequent reads outside the grant. | 17.2, 17.9 |
| Listing exceeds the message budget; stale handle; forbidden write | Return LIMIT_EXCEEDED for the entire list, NOT_FOUND for the closed handle, and fail the mutation locally. | 17.9 |
| Revocation during asynchronous read or listing, before reply or local delivery | Recheck the instance at completion, fail pending work and discard late results. Bytes already delivered cannot be recalled. | 17.3, 17.10 |
| Busy descriptor, cached pages, missing CLOSE, or new mount at the old path | Disable the revoked view immediately on observation, release Client handles and discard old callbacks. Native removal may finish later. | 17.9–17.10 |
| Idle mount, stalled partial CALL, migration, or silent partition | Healthy idle needs no operation timer. Partial calls have a progress deadline. Migration preserves the binding; detected loss tears it down. Reconnect needs new consent. | 17.7, 17.10 |
| Server helper dies or backend cannot isolate or disconnect the view | Refuse installation if isolation or teardown cannot be enforced. On backend death disconnect the view and release the Client grant on detected stream loss. | 17.8, 17.10 |

The walkthrough identified conflicting base and extension rules for action 3 and for setup-caller exit after mount handoff. It also exposed gaps between selection and pinning, consent and object lookup, and asynchronous completion and revocation. Draft-54 resolves these at the relevant transitions. The review establishes expected protocol outcomes. Native Session isolation, cache invalidation, handle confinement and delayed unmount still require platform evidence during RF.

### Combined RF implementation review (0.4.5-1)

On 2026-11-24 the program review traced all 24 rows through RF001–RF004. The Linux shared-Session test distinguishes ambiguity, a busy granting Attachment and a second Attachment without reverse access. QUIC tests cover a mismatched Session, an interrupted local handshake, cancellation during create and migration of an open mount Channel. A real FUSE regression suspended the Client worker with an open-descriptor read outstanding, then killed the Server worker. Before the fix consent remained active past the test deadline. The Server now observes worker exit independently of progress on the Channel, fails pending work and releases the Client grant. The row-by-row evidence and platform limits are in `stpt/docs/RF_REVIEW.md` in the program repository. These corrections implement draft-58.

### RF003 implementation checks (0.4.4-0)

The native Windows and Linux Session tests run the CLI through an authenticated Server-opened Channel with a scripted local approver. They cover multi-frame read and create, refusal and no-replace publication. A Linux pseudo-terminal test also runs the actual consent command, enters the fresh challenge, then verifies that reusing it for a second request is refused. The reverse Channel tests exercise a stale Attachment, unsolicited START, consent and START deadlines, a stalled create, caller abandonment, binding loss and loss of the final create result. Digest and offset cases cover empty content, incomplete or overlapping transfers and integrity failure. Mount lifecycle validation is recorded below.

### RF004 implementation checks (0.4.5-0)

The Linux native test uses a real FUSE device, QUIC Connection, Session shell and a second local consent terminal. It reads and lists the selected tree, attempts a write and link traversal, holds a readable descriptor and a busy working directory, and checks isolation from another OS session running as the same Account. It exercises local revocation, refusal, abandoned setup, detach, Session termination, Connection loss, worker death and abrupt Server exit. Both shared and private mappings are checked for failure to populate pages. The same filesystem operations run from an ordinary login Account in a separate case. A second Attachment can detach without revoking the granting Attachment. Unit tests cover pathname replacement, a file moved outside the pinned root, nested mounts, stale handles, invalid request fields and oversized listings. QUIC tests send MOUNT_READY across a detach and a partial CALL after an idle interval longer than its progress timeout. The cancelled setup returns no path, and only the stalled call times out.
