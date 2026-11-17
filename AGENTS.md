This document answers for questions:"How to work with a repository?"
Read docs\RULES.md before modifying the project.

# 1	Do not:
- change architectural decisions from RULES.md;
- renumber roadmap items;
- silently remove roadmap items;
- introduce dependencies without justification;
- implement functionality belonging to a later Tier unless requested;
- weaken security to simplify implementation;
- add overly detailed descriptions for items.

# 2	When completing a roadmap item:
1. implement it;
2. add/update tests (skipped for documentation-only items);
3. update ARCHITECTURE.md and/or PROTOCOL.md;
4. bump the program version (RULES.md 1.6) if program code changed;
5. mark the item completed in the ROADMAP.md of its track.

P items (protocol) skip steps 2 and 4; their result is PROTOCOL.md and DECISIONS.md.

## 2.1	Documentation synchronization

After every change, review the documents below and update them when the change affects their contents.

stpt-protocol (protocol, CC0 1.0). `docs\` holds project files, `specs\` holds the specification:
- docs\RULES.md - fundamental project decisions (constraints, see 2.1.4)
- docs\ROADMAP.md - protocol roadmap (P items)
- docs\GATES.md - registry of gates
- specs\PROTOCOL.md - STPT protocol specification
- specs\DECISIONS.md - design decisions and open questions; update when a question is decided or reopened
- README.md - front page of the protocol repository

stpt (program, GPLv3):
- docs\ARCHITECTURE.md - architecture of the program code (for developers)
- docs\ROADMAP.md - realisation roadmap (R items)
- README.md - user documentation, front page of the program repository

### 2.1.1	ROADMAP.md
Update the status of implemented or changed roadmap items.
Never renumber existing roadmap item IDs.
If a roadmap item changes in substance or structure, the existing item is marked as completed with a ❌ symbol and the date in YYYY-MM-DD format. Afterwards, a new item is added to this or another section, with the corresponding number, description, and date of addition.
❌ - mark of cancelled item
➕ - mark of addition
✅ - mark of implementation
Each mark is accompanied by a date.
Items of the initial roadmap don't contain any addition marks.

### 2.1.2	ARCHITECTURE.md
Keep the document synchronized with the actual implementation.

Update it whenever a change affects:
- component boundaries;
- Connection or Session lifecycle;
- network transport;
- authentication or cryptography;
- client/server responsibilities;
- persistent state;
- platform-specific behavior;
- important dependencies.

ARCHITECTURE.md must describe the current architecture, not planned or obsolete architecture.

### 2.1.3	PROTOCOL.md
Describes the STPT wire protocol: messages, streams, handshake, negotiation, Connection and Session semantics.
Update it whenever a change affects protocol structure.
The specification may run ahead of the implementation; mark unimplemented parts as draft.

### 2.1.4	RULES.md
Treat project rules as constraints.
Do not modify RULES.md merely to make an implementation conform to the code.
