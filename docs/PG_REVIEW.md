# Tier PG specification review

2026-11-25, draft-59. This is a document/state-machine walkthrough of PG001–PG004, not a program test run. The [normative draft](../specs/PROTOCOL.md#18-port-forwarding-pg001pg004-draft) supplies the wire format and the outcomes below. At that specification review RG001–RG004 were pending. These rows are acceptance criteria for their implementation on both primary platforms.

| Scenario and event order | Required outcome | §18 |
|---|---|---|
| Capability absent, flow before authentication, Server-opened rule, or unselected mode | Connection protocol violation; no socket work. | 2 |
| Malformed offer; selection enlarges maxima, adds modes or duplicates them | Ignore malformed offer; reject invalid selection as protocol violation. | 2 |
| Authenticated Account can run a shell but has no forwarding authorization | Deny forwarding; authentication and terminal permission do not enable it. | 8 |
| Local forwarding without any terminal or Session | Rule acceptance precedes Client listener activation; fixed target is reached from Server. | 1, 4–5 |
| Remote forwarding with locally configured Client target | Server listener dispatches only to that target; target field on a remote flow is forbidden. | 4–5 |
| Server names an arbitrary rule ID or a rule from another Connection | Channel-local rejection, no local target lookup/connect and no fallback. IDs are not transferable authority. | 5 |
| Remote flow arrives before READY because its stream overtakes the rule stream | Bounded wait under the pending rule's deadline; no target contact until successful READY validation. | 4 |
| Rule refused or Client cancels while an early remote flow waits | Invalidate rule and reject/discard queued flows; no target contact. | 4 |
| Listener setup fails or requested nonzero port is already in use | Failed READY or local cancellation; no silent port/address substitution. | 3–4 |
| Remote port 0 receives an actual port outside policy | Close provisional listener before accepting, return refusal. | 8 |
| IPv6-only loopback request on a dual-stack host | No IPv4 exposure; require a separate IPv4 rule. | 3 |
| Non-loopback or wildcard requested by local user without administrator remote-bind permission | Refuse remote rule. Both local intent and Server permission are required. | 3, 8 |
| Explicit remote ingress restriction rejects a source | Close accepted socket before opening a flow to Client. | 8 |
| Terminal detaches or Session exits while Connection remains live | Forwarding continues independently; ending the rule or Connection closes it. | 1 |
| Connection migrates, then is lost and a new one authenticates to the surviving Session | Migration preserves sockets; loss destroys them. New Connection must create new rules; no replay/resume. | 1 |
| Rule FIN races with DNS completion, TCP connect or CONNECTED delivery | Invalidate first, cancel tasks, close late sockets and discard undelivered data; no automatic retry. | 4–5 |
| One rule is revoked while another serves terminal-independent traffic | Close that rule's listener and all its flows; other rule/terminal/Session remains usable. | 4 |
| One flow resets while its rule stays authorized | Abort both socket halves and discard queues; subsequent independent flows remain possible. | 6 |
| Target refuses connect or DNS fails | Failed CONNECTED, FIN exchange, no DATA; SOCKS maps the reported outcome to failure. | 5, 7, 9 |
| Duplicate READY/CONNECTED, DATA before success, wrong direction or malformed known field | Live-stream protocol violation. Cancelled-stream late messages are discarded instead. | 4–6 |
| Sender reads EOF after DATA while reverse TCP direction still produces output | Drain DATA then FIN; receiver drains bytes then shuts down only socket write half; reverse traffic continues. | 6 |
| Both directions are empty, or a FIN arrives halfway through a frame | Empty directions after success complete cleanly; incomplete frame aborts, never clean EOF. | 6 |
| TCP reset or STOP_SENDING with pending writes | Abort entire flow; do not turn reset into graceful EOF or claim application receipt. | 6 |
| One flow blocks on write; another carries data and control cancels the first | Bounded backpressure and independent control/cancellation; blocked progress expires without affecting other flows. | 6–7 |
| Healthy idle flow lasts longer than 60 seconds with Connection keepalive | No data-progress timeout at a clean frame boundary with no pending bytes. | 7 |
| Peer trickles a partial opening, SOCKS handshake or DATA frame | Total opening/handshake or bounded stalled-progress deadline applies; no indefinite resource retention. | 7, 9 |
| Stream credit exhausted before opening, or peer never returns READY/CONNECTED | Local opening deadline aborts affected attempt and releases its socket/reservation. | 7 |
| Concurrent opens exceed selected flow/rule maxima; closed sockets are still reaping | Atomic aggregate reservation includes opening/teardown; fail excess admission without unbounded queueing. | 2, 7 |
| Multiple Connections for one Account exhaust aggregate tasks, sockets or bytes | Enforce Account/global budgets independently of per-Connection negotiation; pending DNS and queued native I/O count. | 7 |
| DNS name passes name rules but returns forbidden loopback/private IP or mixed A/AAAA | Check each numeric candidate; never probe forbidden addresses. Connect only a checked allowed address, without second resolution. | 3, 8 |
| Allowed name changes resolution on next flow; IPv4-mapped address attempts ACL bypass | Recheck new candidates and normalize mapped IPv4; no cached authorization of an unchecked address. | 3, 8 |
| Numeric/DNS address is malformed, scoped, multicast or unspecified | Malformed wire fields violate the protocol; well-formed unsupported destinations fail UNSUPPORTED without fallback. | 3 |
| Privileged Server could bind/connect beyond the Account's native network authority | Enforce Account authority or refuse; service privilege is not delegated by enabling forwarding. | 8 |
| Administrator revokes a rule/destination while asynchronous completion is pending | Enforce on local observation, discard late work and abort affected sockets without peer acknowledgement. | 4, 8 |
| Administrator wants to enforce Server-side destination filters on remote targets | Remote delegation cannot supply that guarantee; disable remote mode if this is required. | 8 |
| SOCKS5 CONNECT for IPv4, IPv6 or ASCII domain | One flow per request; domain resolves on Server. Send success only after target connection, with target-side local socket as BND endpoint. | 5, 9 |
| SOCKS client offers no supported method, requests BIND/UDP or unknown address type | Method FF / REP 07 / REP 08 respectively, then close; no forwarding side effect. | 9 |
| SOCKS client pipelines payload before CONNECTED | Buffer only within limits; withhold DATA until success. Never relay SOCKS headers to target. | 7, 9 |
| SOCKS listener exposed beyond loopback without ingress enforcement | Refuse configuration. Explicit exposure acknowledgement and local ingress allowlist are required. | 9 |
| SOCKS application resolved its name before contacting the proxy | Forward numeric target as given; do not claim the proxy prevents application-local DNS leaks. | 9 |
| Target accepted connection before cancellation, but success was lost | Target may have observed connection; no exactly-once guarantee and no automatic retry. | 5–6 |

The review checked allocation uniqueness against capabilities 1–3, framing against §7, authentication/stream opening against §8 and request-independent service lifetime against §15. It identified the cross-stream remote-open race, ephemeral-port authorization, cancellation of late connect completions and the difference between idle sockets and blocked work; the draft defines each explicitly. It also distinguishes restricted SOCKS5 CONNECT support from full RFC 1928 conformance.

Implementation validation must exercise actual Windows/Linux socket authority, IPv6-only binds, partial I/O, native DNS cancellation and bounded cleanup in addition to QUIC fault peers. This review supplies no evidence that those program behaviors exist yet. ARCHITECTURE, program version, realisation completion status and existing platform gates are unchanged.

## RG implementation review (0.5.0-0)

RG001–RG004 are implemented in program 0.5.0-0. The [implementation summary](RG_REVIEW.md) records native/QUIC validation and limits. The program repository contains a 41-row mapping distinguishing runtime tests from inspection; the draft-59 table above remains the original acceptance criteria.

## Full RG scenario review (0.5.0-1)

The [full implementation review](RG_REVIEW.md) replaces the earlier inspection-only gaps with authenticated, native, fault-peer and controlled backend evidence. The original 41 draft-59 acceptance criteria remain unchanged; draft-61 records the fixes without wire changes.
