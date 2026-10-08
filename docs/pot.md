# Pot: architecture plan

Status: design sketch, decisions recorded 2026-10-08 by Roman. Full design doc: https://claude.ai/code/artifact/5023b6dc-bb48-474d-853a-88976013e450

## What a pot is

A pot is a self-contained node that runs its own runtime, offers a set of skills, and talks to other pots over one messaging layer. The messaging layer can use HTTP, TCP, UDP, peer-to-peer links, or any of the adopted transports. Each pot has a keypair, and its public key is its address.

## Decisions

| Question | Decision |
| --- | --- |
| Who writes skills? | Both. Built-in skills are native modules. Outside code runs as WebAssembly components in a sandbox. |
| Where do pots run? | Everywhere, from servers down to embedded boards. One core, with optional features dropped on small devices. |
| Implementation language | Rust. |
| Can a pot ship skill code to a peer? | Call-only for now. The manifest format stays ready for signed code shipping later. |
| Discovery | Fully peer-to-peer (static list, mDNS, DHT). Any pot may optionally act as a rendezvous server that others opt into. |
| Trust model | Private groups by default. Chosen skills can be exposed to an open network. |
| State | The runtime is stateful: keys, manifest, peers and routes, tokens and revocations, rate-limit counters, mailbox queues. Skills are stateless. |

Terminology: "skill" is the function a pot offers (e.g. `image.resize@1`). A "grant" is a signed token that allows a peer to call specific skills.

## Layers

| # | Layer | Responsibility |
| --- | --- | --- |
| 1 | Skills | Functions the pot offers, native or WebAssembly |
| 2 | Host interface | Controlled access to the device (files, sensors, GPU, pins). Outside code gets only what this layer grants. |
| 3 | Runtime | Loads, isolates, supervises and upgrades skills |
| 4 | Storage | Runtime data: keys, manifest, peers, grants, rate limits, mailbox queues |
| 5 | Identity and security | Keys, grants, permission check on every call |
| 6 | Messaging | Signed envelopes, request/reply matching, delivery needs |
| 7 | Routing | Best-path choice and relaying through other pots |
| 8 | Transports | HTTP, TCP, UDP, P2P and adapters |

Management (local admin API and CLI: install, upgrade, configure, health check) and Observability (logs, traces, metrics) run alongside every layer. Discovery sits underneath and feeds Routing.

Each layer talks only to its neighbours, so a transport or runtime can be replaced without touching the others.

## Transports

Core: HTTP(S), TCP, UDP, P2P (QUIC preferred as the main wire; libp2p is a candidate for the P2P layer).

Adopted adapters: Unix sockets and shared memory, WebSocket and WebTransport, gRPC, WebRTC data channels, MQTT or NATS, Bluetooth LE and Wi-Fi Direct, LoRa or mesh radio, serial, USB and CAN, WireGuard or Tailscale overlay, Tor and I2P, store-and-forward mailbox.

Every skill declares its delivery needs (timeout, at-least-once, idempotent). The messaging layer only uses transports that meet them.

## Security

- Every link is mutually authenticated with pot keys (TLS or Noise).
- Envelopes are signed end to end, so a relay pot cannot read or alter them.
- Grants are signed, delegable tokens, so no central auth server is needed (the UCAN approach).
- Skill names are namespaced under the publisher's key, to avoid collisions on an open network.

## Build plan

First slice:
1. Pot identity: Ed25519 keypair; pot ID is the public key.
2. Manifest format: skills, versions, schemas (protobuf or CBOR).
3. Runtime with native skills only.
4. Messaging over one transport (QUIC or TCP), with mutual authentication.
5. Static peer list for discovery.
6. Demo: pot A calls `echo@1` on pot B, and B calls back.

Then, in rough order:
- WebAssembly skills
- Unix sockets and WebSocket transports
- HTTP and UDP transports
- mDNS discovery
- Grants
- P2P with NAT traversal
- Routing and relaying
- The remaining transport adapters

## Known gaps

1. **No use case yet.** Name one or two concrete scenarios to decide which transports and features come first.
2. **Lite pot for microcontrollers.** A board with kilobytes of RAM cannot host the full stack. Proposed: a lite profile that reaches the network through a gateway pot.
3. **Delivery guarantees differ by transport.** Handled by per-skill delivery needs (see Transports).
4. **Envelope size on LoRa and BLE.** Needs a compact binary envelope and cached session keys.
5. **Native and WebAssembly skills have two interfaces.** Proposed: one interface definition, such as WIT.
6. **Key loss and revocation.** Proposed: key rotation, short-lived grants, and an admin key per private group.
7. **Abuse on the open network.** Proposed: rate limits, invites or proof-of-work to join the DHT, and per-peer quotas.
8. **Phones drop off the network.** Proposed: mobile pots act as clients that reach the network through a relay or mailbox.
9. **Tracing calls across pots.** Proposed: trace IDs in every envelope and a local call log per pot.
10. **Two layers squeezed together.** Identity and security is distinct from messaging; keep them separate in code.
11. **Stateful runtime versus stateless skills.** Resolved: runtime owns state.
12. **Runtime versus routing for multi-hop relays.** Resolved: a dedicated Routing layer.

## Comparable systems

- wasmCloud: Wasm components calling declared providers across a mesh
- libp2p: peer IDs, pluggable transports, hole punching, relays, DHT
- Erlang/OTP: location-transparent messaging, supervisors
- Akka and Orleans: actors addressed by ID
- Dapr: sidecar building blocks for service calls
- Holochain: autonomous peer-to-peer agents without a server
- UCAN: delegable capability tokens

The gap Pot fills is combining these concerns in one unit that runs anywhere. wasmCloud depends on a central lattice, Dapr assumes a cluster, and libp2p stops at the network layer.

## Open for Roman

- Name the first use case.
- Confirm the lite-pot profile for microcontrollers.
- Confirm the Rust repository to start in.

## Decisions from the 2026-10-08 review

Each of these was chosen by Roman on a decision card in the project thread.

- **First use case:** personal devices (laptop, phone, home server over LAN and P2P).
- **Microcontrollers:** deferred. The default is that the first build targets full pots only. Not yet confirmed.
- **Runtime profiles** (`full`, `lite`, `gateway`): fixed at restart, not changeable while running. Not shareable as files for now.
- **Lost key:** no recovery. The pot gets a new identity, and peers re-trust it by hand.
- **Open-network abuse:** per-skill rate limits, plus a small proof-of-work to join discovery.
- **Phones:** relay through the home pot, which holds messages and passes calls through while the phone sleeps. A phone is never a server.
- **Delivery guarantee:** the caller picks per call, never above the ceiling the skill declares.
- **Envelopes on LoRa and BLE:** compact binary format with short IDs. Every call still carries a full signature.
- **Native and WebAssembly skills:** two interfaces for now. WIT is the candidate for later.
- **Skill names on the open network:** readable name (for example `image.resize`), with the publisher's key as a tiebreaker.
- **Debugging:** every envelope carries a trace ID, and each pot on the path logs it.
