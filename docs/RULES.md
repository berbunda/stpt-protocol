This document answers for questions:"Which project decisions are considered fundamental?"
# 1	STPT Project Rules

## 1.1	Licensing
STPT is licensed under GNU GPLv3. - execution program
- local path: "E:\prj\rust\stpt\stpt"
- remote path: "https://github.com/berbunda/stpt"
STPT-Protocol is licensed under CC0 1.0 Universal - documentation of protocol
- local path: "E:\prj\rust\stpt\stpt-protocol"
- remote path: "https://github.com/berbunda/stpt-protocol"

## 1.2	Language
Rust is the primary implementation language for STPT
English is the primary implementation language for STPT-Protocol

## 1.3	Architecture
- STPT is distributed as a single binary.
- The same binary provides client and server functionality.
- Windows 11 amd64 and Debian-based Linux amd64 are primary platforms.
- The secondary platforms are other linux platform, bsd platforms (in future)
- Connection and Session are distinct protocol concepts.
- A Session MUST NOT depend on the lifetime of a Connection.

## 1.4	Terminology

### 1.4.1	Tier
A major roadmap development stage.
The roadmap has two tracks, each in its own file:
- P — protocol: `stpt-protocol/docs/ROADMAP.md`;
- R — realisation (program): `stpt/docs/ROADMAP.md`.

Format:
Tier PA, Tier PB, Tier PC...
Tier RA, Tier RB, Tier RC...

### 1.4.2	Roadmap Item
A development unit belonging to a Tier.

Format:
PA001, PA002, PB001...
RA001, RA002, RB001...

Identifiers are permanent and MUST NOT be reused.
An R item implementing part of the protocol references the P items it depends on and MUST NOT start before they are at least draft.

### 1.4.3	Gate
A runtime, platform, protocol, dependency, or capability
restriction affecting feature availability.

Format:
G1, G2, G3...

Gate identifiers are independent of roadmap identifiers.

## 1.5	Claude Note
You are free to record any information you deem necessary — with the exception of passwords, secrets, and keys — in the NOTES.md file.

## 1.6	Build managements
Every change to the program code must be accompanied by an increment of the program version in the `Cargo.toml` file under the `[package]` section.
The versioning follows this structure: x.y.z-b, where
x - major changes
y - middle changes
z - minor changes
b - bugfixes (builds)
Resetting lower-order parts: when a higher part increases, every lower part is reset to 0, and `-b` is always written explicitly (e.g. 0.5.16-8 → 0.5.17-0, never 0.5.17), so every version stays in the x.y.z-b form and sorts consistently under semver.

## 1.7 Repository managements
You may free make commit and push to repository whenever you deem necessary.

## 1.8	Security
- STPT MUST NOT use custom cryptographic algorithms or primitives; only proven implementations.
- No STPT application data is sent without TLS protection.
- A server MUST NOT access client files without explicit permission granted by the client.