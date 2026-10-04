# Observe Pulsar

### The Assembly — Sovereign Edge Infrastructure

**Overview · Version 3.0 (public) · rev. 2026-10-03**
Liturgy Bureau · liturgy.one

| | |
|---|---|
| **Stack** | `no_std` Rust · ESP32 · WireGuard · Postcard |
| **Status** | Architecture bench-proven. Early stage. |
| **Field deployments** | None yet. The first are planned (§4.2). |
| **Certifications** | None yet. A path, not a claim (§5). |

> **How to read this document.** Each section opens with a short *In plain terms* passage that stands on its own. The technical body follows for those who want it. Section 1 pairs the two directly: §1.1 is the plain account, §1.4 is the engineering one.

**Contents**
1. Overview
2. Market Comparison
3. Design Principles
4. Readiness
5. Where the Architecture Goes
6. Conclusion

---

## 1. Overview

### 1.1 In plain terms

Observe Pulsar is a private, encrypted communication system for physical devices — sensors, switches, motors, cameras, valves — that lets them talk directly to you, and to each other, without any company, cloud, or third party in between.

Most connected devices today work by relaying your commands through someone else's servers: you press a button, the request travels to a data center, gets processed, and travels back. That means your devices only work when the internet works, when the company's servers are up, when their business model still supports your hardware. You don't own the connection; you rent it.

Observe Pulsar takes that middleman out of the path.

Each device creates its own encrypted, private tunnel — a WireGuard link. By default the tunnel runs to a relay server that you operate; whenever a direct path exists, devices connect straight to your phone, your computer and each other. The encryption keys are generated on your own phone or computer and written to each device over Bluetooth. The private keys never leave your devices: not the relay, not the network provider, and not the system's creators ever hold them.

The relay is not a company's server. It is a small machine you run yourself, and that matters for a plain reason: in the default configuration the relay passes traffic between your devices and can see it in transit. On a direct link, only the two devices involved can read it. This is one of the reasons the direct paths are a priority (§3.7).

Devices are given a role at setup — Observer (measures), Actor (switches), Flow (regulates), Kinetic (moves), Vision (sees), Link (bridges), Intent (decides) — and the device's own firmware enforces it. The pins are assigned when the firmware is built, so a sensor cannot accidentally trigger a motor, and a valve controller cannot read a camera.

Different users can share the same relay while staying invisible to one another, separated at the operating-system kernel level.

There are no cloud accounts, no subscriptions, no mandatory updates, and no telemetry collected by the builder. The system is provisioned once over Bluetooth, and from then on the devices are yours: you hold the keys and you run the relay. If the relay server goes down, devices that can reach each other directly keep talking. (Section 4 states exactly what is proven today and what is still being built.)

The firmware, the server, the shared protocol and the core of the app are all written in Rust, speaking one binary protocol across every layer. It runs on inexpensive, widely available microcontrollers, in enclosures designed for the job.

In short: Observe Pulsar is infrastructure that its owner controls, built on open, well-studied cryptography, for people and places that cannot or would rather not depend on someone else's servers.

### 1.2 The problem it answers

Much of today's networking and IoT equipment assumes things that do not hold in many parts of the world:

- **That you have stable power.** Try that on a research base running on solar and batteries through a polar night.
- **That you have a reliable internet connection.** Try that on a ship, or after an earthquake.
- **That connecting to "the cloud" is even allowed.** Some places legally cannot send data outside their own borders. Some have no cloud to reach in the first place.
- **That someone, somewhere, is managing it all centrally.** There is no help desk in Antarctica.
- **That there is unlimited compute and memory to throw at the problem.** These chips have a few hundred kilobytes of memory.

Observe Pulsar starts from the opposite assumption: none of that is available. It is designed for the conditions that actually exist in those places, not the ones most products assume.

### 1.3 The Assembly at a glance

Observe Pulsar is not one program. It is an **Assembly**: a small set of parts, built together, that share one protocol and one set of design principles.

| Part | What it is, in plain terms | Status |
|---|---|---|
| **Pulsar node** (`pulsar-object-01`) | The small device that senses or acts. Bare-metal firmware on a coin-sized ESP32 chip, with no operating system underneath. | Bench-proven (ESP32-C3) |
| **Conductor** | The app on your phone or computer. Sets up new devices over Bluetooth, controls them, shows their data. | Bench-proven |
| **Registry & relay** | A small server you run yourself. Remembers each device's identity, forwards traffic between your devices, keeps tenants apart. | Bench-proven |
| **Router** (`pulsar-router`) | The Assembly's own gateway: WiFi, addressing, routing, and an optional cellular link. It replaces the ISP router. | In build |
| **Protocol** (`pulsar-protocol`) | The single binary language every part speaks. | Bench-proven |

Three ideas carry most of the design:

- **Two paths, always.** Devices connect directly to each other first. If that is blocked, which happens often on real networks, they fall back to the relay. The relay is a fallback, not a dependency.
- **Setup without the internet.** A new device is configured from your phone over Bluetooth. No account, no cloud console, no connection required.
- **One job per device.** Each device carries a role, and that role decides which pins it may touch and which commands it will accept.

**The router.** With `pulsar-router`, the Assembly brings its own gateway: it advertises its own WiFi network, assigns addresses, performs routing and address translation, and can reach the wider world over a cellular link. No ISP router is needed in the loop. The one external dependency that remains is the carrier SIM, and it matters only when the Conductor is outside the local mesh.

### 1.4 Technical overview

*For engineers. Readers who prefer the plain account can continue at Section 2.*

Observe Pulsar is a vertically integrated, self-hosted IoT stack that establishes mutually authenticated, encrypted WireGuard tunnels between resource-constrained edge nodes, a user-operated relay and a user-controlled Conductor, with no cloud intermediary and no centralized trust anchor.

#### Protocol & Identity

- **Noise_IKpsk2 handshake (WireGuard)** with X25519 key agreement, ChaCha20-Poly1305 AEAD and BLAKE2s hashing. Implemented in `#![no_std]` Rust on the firmware, `boringtun` (userspace) in the Conductor, and kernel WireGuard on the relay.
- **Two trust modes.** In the default star topology, each node holds a WireGuard session with the relay. The relay decrypts a packet, re-encrypts it for the destination peer and forwards it, so it can read traffic in transit, and the silo rules are enforced on the decrypted packet. In a direct P2P session, two peers hold a session with each other's public keys and the relay forwards only ciphertext it cannot decrypt. P2P is partially working (§4), so the star path is still the default and the fallback.
- **Postcard** binary serialization, shared identically across firmware, server and app: one schema, no translation layer, no schema drift.
- **Provisioning over BLE GATT.** The Conductor generates the node's X25519 keypair on-device (X25519 is used for all WireGuard and provisioning keys), registers the node with the registry, and receives an assigned IP and the server public key. It then assembles a `ProvisioningPayload` (keys, endpoint, WiFi credentials, role, safe-state configuration), writes it over BLE, and the node reboots into the mesh.

#### Firmware (`pulsar-object-01`, ESP32-C3; S3 port in progress)

- **Runtime and stack:** Embassy async runtime; `smoltcp` userspace TCP/IP over a `VirtDevice` backed by the WireGuard tunnel.
- **Custom WireGuard (`wg.rs`):** full Noise_IK state machine (initiator and responder), 64-bit sliding-window anti-replay, proactive rekey (120 s), key rotation that preserves in-flight packets.
- **Dual-session architecture:** a persistent relay session plus an opportunistic P2P session. A direct session forms when endpoints are discovered through a LAN candidate carried in TCP responses. *Partially working; see §4.*
- **Role system (`0x00`–`0x07`):** Lab, Observer (BME280 / generic I²C), Actor, Flow (soft-PWM), Kinetic (4-byte `0x4B` streaming packets at 25 ms, 2 s lease), Vision, Link, Intent. Pins are assigned at compile time and the firmware refuses commands that contradict the node's role. Vision, Link and Intent are stubs.
- **Safe-state contract:** a watchdog monitors `LAST_APP_HEARTBEAT`; on lease expiry, GPIO is driven to the configured fail-safe level.
- **Two-phase boot:** Phase 1 is 120 s of strict WiFi validation, wiping on failure. Phase 2 writes a tombstone (`0x414C4956`) and retries indefinitely, for power-loss resilience.
- **Time:** SNTP feeds TAI64N timestamps for WireGuard replay protection; the offset is held in a `critical_section::Mutex<Cell<u64>>`.

#### Router (`pulsar-router`, ESP32-S3 + SIM7600A) — *in build*

- Hand-written PPP/HDLC framer (`ppp.rs`) with LCP/IPCP state machines over UART, exposed as a `smoltcp::phy::Device` for the cellular WAN.
- Hand-written IPv4 NAT (`nat.rs`): TCP/UDP/ICMP tracking, port allocation, garbage-collection timeouts (TCP 300 s, UDP 60 s, ICMP 30 s).
- WiFi AP (`192.168.8.1/24`), DHCP, and an HTTP provisioning server on the AP interface.
- WAN switching by explicit operator action (`WAN:CELLULAR` / `WAN:WIFI` / `WAN:NONE`). There is deliberately no auto-failover.

#### Registry Server (`pulsar-registry-server`, Axum + SQLite)

- **IP allocation:** a 12-bit Silo ID maps to a /20 per tenant (4,095 silos × 4,096 IPs). Conductor = `10.{second}.{base}.1/8`; Pulsars = `10.{second}.{base+1+offset}.{offset}/8`.
- **Kernel-enforced isolation:** one `ipset` (`PULSAR_SILOS`, `hash:net,net`), one `iptables -I FORWARD -i wg0 -o wg0 -m set --match-set PULSAR_SILOS src,dst -j ACCEPT` rule, and a default DROP.
- **Bootstrap sync:** on startup the server reads the database, runs `wg set wg0 peer <pubkey> allowed-ips <cidr>` and `ipset add`, and writes `/etc/wireguard/peers.conf`.
- **WAN endpoint tracking:** periodic `wg show wg0 endpoints` updates the database for hole-punching.

#### Conductor App (`observe-pulsar`, Tauri + Angular)

- **Userspace WireGuard and TCP/IP:** `boringtun` and `smoltcp` run in a Tokio task (`network_worker.rs`). No TUN/TAP, no root, no OS VPN stack.
- **Virtual device:** a `VirtDevice` feeds the `smoltcp::Interface` with 8 concurrent TCP sockets over the encrypted tunnel.
- **P2P mesh logic:** per-peer `boringtun` sessions in a `HashMap`; direct handshake when a LAN/WAN candidate is discovered; 120 s rekey; 10 s hole-punch probes.
- **AllowedIPs enforcement:** every outbound TCP connection is checked against server-issued CIDRs before a socket is created, so tenant isolation also holds in userspace.
- **Kinetic control:** a virtual joystick streams 4-byte packets (`0x4B`, x, y, flags) over a persistent TCP connection at 40 Hz.

#### Hardening path & Governance

- **Target: IEC 62443-4-2 SL2. Nothing here is certified yet; this is the engineering path.** Secure Boot v2 (eFuse), Flash Encryption, encrypted NVS, a signed-firmware pipeline (`espsecure.py`), an SBOM (CycloneDX), and `cargo-audit` / `cargo-deny` CI gates.
- **ADR-governed architecture:** five-plus Architectural Decision Records covering flash layout, silo sizing, NVS segregation, the provisioning trust model, and tombstone mirroring.
- **Role promotion gate:** an FMEA template plus CI enforcement before any role leaves stub status.
- **Covenant Protocol:** the project's written design principles (the Zero-Extraction Rule, the Non-Transactional Constraint and Operational Stewardship), reflected in the architecture and not merely documented (§3).

In summary: a single-protocol, single-language, full-vertical IoT stack in which identity and access are handled by cryptographic keys that the owner holds, with no cloud account or vendor in the loop.

---

## 2. Market Comparison

### In plain terms

Most existing systems cover one or two pieces of the puzzle and leave the rest to other tools. Some give you a polished app but no control over the device. Some give you control of the device but no secure way to reach it. Some give you a secure tunnel with no device at the end of it. Observe Pulsar is built to cover the whole chain, from the chip to the phone in your hand, and, with the router now in build, the gateway in between.

### 2.1 The four layers, plus the gateway

Every IoT system operates across four architectural layers. Observe Pulsar is built to cover all four, and adds a fifth piece: its own gateway.

| Layer | What it means | Observe Pulsar | Status |
|---|---|---|---|
| **1. VPN / isolation** | Encrypted tunnels and tenant isolation | WireGuard + kernel `ipset` silos | Bench-proven |
| **2. Device firmware** | Bare-metal firmware the owner controls, with a cryptographic identity | `#![no_std]` Rust + custom WireGuard | Bench-proven |
| **3. Provisioning** | Device onboarding with zero internet | BLE GATT, phone-generated keys | Bench-proven |
| **4. Controller app** | Native WireGuard client and role-aware UI | Tauri/Angular + `boringtun` + `smoltcp` | Bench-proven |
| **+ Gateway** | The network edge itself: WiFi, routing, NAT, cellular WAN | `pulsar-router`: hand-written PPP, NAT, AP, DHCP | In build |

### 2.2 Capability comparison

**✅** yes · **◐** partial, or with caveats · **❌** no. Every row is phrased so that ✅ is the better outcome for the person who owns the devices.

| Capability | Observe Pulsar | Home Assistant | Tasmota / ESPHome | Tailscale | Balena | AWS / Azure IoT | Consumer IoT |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Encrypted tunnel as a core feature | ✅ | ◐ | ❌ | ✅ | ◐ | ❌ | ❌ |
| Tenant isolation enforced in the kernel | ✅ | ❌ | ❌ | ◐ | ❌ | ◐ | ❌ |
| Firmware you own and can audit | ✅ | ❌ | ✅ | ❌ | ◐ | ❌ | ❌ |
| Cryptographic identity per device | ✅ | ❌ | ◐ | ◐ | ◐ | ✅ | ❌ |
| Role-aware behavior inside the device | ✅ | ◐ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Complete offline onboarding | ✅ | ❌ | ◐ | ❌ | ❌ | ❌ | ❌ |
| Controller app with a native tunnel client | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Peer-to-peer mesh with relay fallback | ◐ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Fail-safe behavior defined per role | ✅ | ❌ | ◐ | ❌ | ❌ | ❌ | ❌ |
| Keeps working when the central service is gone | ◐ | ◐ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Own gateway that replaces the ISP router | ◐ | ❌ | ❌ | ❌ | ❌ | ❌ | ◐ |
| No vendor-held data or telemetry path | ✅ | ◐ | ✅ | ◐ | ❌ | ❌ | ❌ |
| No subscription | ✅ | ✅ | ✅ | ◐ | ❌ | ❌ | ◐ |
| No vendor lock-in | ✅ | ◐ | ✅ | ◐ | ❌ | ❌ | ❌ |
| Whole vertical in one language and one protocol | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Typical hardware cost per node | ~$8–15 | varies | ~$8–15 | n/a | ~$35+ | varies | $30–300+ |

A few cells deserve a word. For Observe Pulsar, the peer-to-peer mesh and server-independent operation are ◐ because both are partly working today (§4). The router is ◐ because it is in build. For the others, ◐ usually means "possible through add-ons, optional configuration, or a different layer of the stack."

*Assessments of other projects describe their default, documented configurations as understood at the time of writing. They are good projects with real strengths, several of which Observe Pulsar does not match: Home Assistant's breadth of integrations and Tailscale's mesh maturity are two. Corrections are welcome.*

### 2.3 What each has, and what each is missing

| Player | Strengths | Where Observe Pulsar differs |
|---|---|---|
| **Home Assistant** | A very good UI and a large integration ecosystem | Native tunnel, kernel tenant isolation, firmware of its own, offline BLE onboarding |
| **Tasmota / ESPHome** | Open firmware on inexpensive hardware | Encrypted tunnel, provisioning infrastructure, role system, identity management |
| **Tailscale** | A mature, widely used WireGuard mesh | Device firmware, provisioning, role model, controller app |
| **Balena** | Fleet management for Linux containers | Cloud independence, kernel silos, BLE onboarding, role model |
| **AWS / Azure IoT** | Very large scale and managed services | Independence from a vendor cloud, firmware you own, peer-to-peer mesh |
| **Consumer IoT** (Alexa, Nest, Ring) | Polish, voice, ecosystem | Ownership: the cloud, the data and the hardware all belong to someone else |

### 2.4 Where it fits

Observe Pulsar is designed to cover all four layers in one system. It has a VPN and isolation layer with kernel-enforced tenant separation. It has device firmware with a cryptographic identity and role-aware behavior. It has a provisioning system that works over Bluetooth with no internet connection. And it has a controller app that runs the WireGuard client itself, rather than relying on the operating system's VPN stack. As far as we know, no other single system combines all four.

The fifth piece is the gateway. With `pulsar-router`, the Assembly can supply its own: no ISP router has to sit between your devices and the wider network. That piece is still in build. The only external dependency that remains is the carrier SIM, if you choose cellular.

Most existing systems cover one or two layers and treat the rest as integrations. Observe Pulsar was built bottom-up to cover all of them, because the layers depend on one another. Device identity needs a tunnel you control. A tunnel needs isolation between tenants. Provisioning with no internet needs keys generated on the phone. A controller that does not rely on the OS VPN stack needs userspace WireGuard and TCP/IP.

This is not a replacement for Home Assistant or Balena; it solves a different problem. We call the category sovereign edge infrastructure: infrastructure at the edge of the network that its owner controls end to end.

---

## 3. Design Principles

### In plain terms

Most systems are easiest to describe by what they add. This one is easier to understand by what it deliberately leaves out: the cloud broker, telemetry collection, the subscription and vendor lock-in. Those choices are built into how the system works, so that you do not have to rely on a promise. The trade-off is that you take on responsibility for running your own system. This section explains the reasoning behind those choices and what they ask of you.

### 3.1 What the system leaves out

Every system starts with choices about what it will not do. Observe Pulsar leaves out the cloud broker, telemetry collection, the subscription layer and vendor lock-in. It also avoids the ready-made option at several layers: no MQTT, no managed BLE frameworks, no pre-built WireGuard libraries on the firmware, no AWS IoT Core, no BalenaCloud, and no OS VPN stack in the app.

The aim is not minimalism for its own sake. It is to make independence a property of the design, enforced by cryptography and kernel isolation rather than by policy. The cost is more work for the builder and for the operator (§3.9).

### 3.2 Naming conventions in the codebase

The codebase uses a small set of consistent terms in logs and documentation. They are practical labels, not branding:

| Term | Where it lives | Meaning |
|---|---|---|
| **Sincere** | Log output (`Sincere Log: ...`) | Marks log lines that report the system's actual state |
| **Consonance** | WireGuard handshake log (`CONSONANCE ACHIEVED`) | Both sides of the handshake hold identical derived keys |
| **Tender Cut-Off** | Actor role safety contract | When a lease expires, GPIO is driven to the configured fail-safe level |
| **Librarian of Truth** | Registry Server | Stores each device's identity (role, key, silo) so it can be restored on a new Conductor |
| **Covenant** | `COVENANT_PROTOCOL.md` | The project's written design principles (§3.3) |

Shared terms make logs and discussions easier to read, and a log line that states what the system is actually doing is more useful than one that states what it hopes.

### 3.3 The Covenant: design principles built into the system

`COVENANT_PROTOCOL.md` states a small set of design principles. Where possible, the architecture implements them, so they do not depend on anyone's good intentions:

| Principle | How the architecture reflects it |
|---|---|
| **Zero-Extraction Rule**: "We facilitate the signal; we do not own the data" | No vendor cloud in the path. The relay is operator-owned. Postcard messages carry no metadata fields. No analytics endpoints. |
| **Non-Transactional Constraint**: "Prioritize client's control over recurring revenue" | No subscription layer, no licensing mechanism, no API rate limits tied to billing. |
| **Rule of Utility**: "If the tool ceases to serve the user's autonomy, the user is encouraged to disconnect" | The `CLRMEM` command wipes NVS flash and returns the device to a virgin state. |
| **Tender Cut-Off**: "If a Pulsar generates system noise, it is your duty to recalibrate or prune the node" | The safe-state lease contract (watchdog, then fail-safe GPIO) and `CLRMEM`. |

The principles include their own exit: a user can wipe any device and walk away at any time.

### 3.4 Who runs the system

Observe Pulsar assumes that the person who uses the system also runs it. The Covenant calls this person the Steward. In practice that means:

- You are responsible for the integrity of your encryption keys.
- You are responsible for the security of your local nodes.
- You are responsible for how your nodes behave. A node that misbehaves is yours to recalibrate or remove.

Commercial IoT products usually take on that complexity for you. That is convenient, but it also means dependencies are accepted, and data and control are ceded, with little visibility. Observe Pulsar takes the opposite approach: the complexity and the control both stay with the owner.

Today the onboarding path asks for a VPS setup, a registry deployment, an app build, a firmware flash, BLE provisioning and key management: ten-plus steps with several points of failure. The Covenant is direct about it: the project will not simplify things for people unwilling to learn how the connection works. The intent is not to exclude anyone. It is a practical filter, because a system whose operator holds the keys needs an operator who understands what that involves.

### 3.5 Why not the cloud

Cloud-connected devices send their data through infrastructure that someone else owns. Whatever a vendor's policy says today, that arrangement makes some outcomes possible, and policies change:

- Thermostat → vendor cloud → a behavioral data point that can be analyzed, shared or sold.
- Security camera → data center → a legal asset of a corporation, subject to law-enforcement obligations.
- Smart speaker → every recognition attempt retained as data that belongs to someone else.

Observe Pulsar takes the third party out of the path by design. There is no vendor cloud holding the data, no vendor account to be compelled, and no business model that depends on your telemetry. Direct device-to-device links can be read only by the two endpoints. Relayed traffic passes through a server you operate, not one a company operates, and in the current star topology that server can read it in transit. That is a different kind of assurance from a privacy policy, although it is not a substitute for an independent security review (§5.4).

Some users have particular reasons to want this:

- Journalists in surveillance-intensive environments who control physical equipment.
- Medical clinics in regions with unreliable internet and active government monitoring.
- Community-run projects building their own physical infrastructure: off-grid settlements, cooperative workshops, community-owned land.
- Remote agricultural sites, where cloud-dependent irrigation can become a crop-loss risk when the connection fails.

For these users, a system that provisions over BLE with no internet, runs encrypted tunnels through a relay they control, and never hands key material to a third party is a good fit for the situation.

### 3.6 Roles: devices that know what they are

In most commercial IoT, a device does not know what it is. A smart plug does not know it is a smart plug; the meaning lives in the cloud, the app and the manufacturer's database.

In Observe Pulsar, a device's role is part of its provisioning data. The `pulsar_role` field is stored in NVS flash. At boot the firmware starts the task for that role, locks the matching pins to it, and refuses commands that contradict the role. The role is tied to the device's WireGuard key and archived by the Registry Server, so it can be restored.

| Role | What it does |
|---|---|
| **Observer** | Reads sensors and reports; does not act |
| **Actor** | Switches things on or off; falls back to a fail-safe state if its lease expires |
| **Flow** | Regulates continuously, for example by PWM |
| **Kinetic** | Drives motion under a 2-second lease |
| **Vision** | Captures images (stub) |
| **Link** | Bridges other protocols into the mesh (stub) |
| **Intent** | Makes local decisions autonomously (stub) |

When a new Conductor is provisioned, it downloads each device's record from the registry and shows the right interface automatically. Identity travels with the cryptographic keys, not with a particular phone. Because roles are enforced by how the firmware is built, a mismatched command is refused rather than carried out.

### 3.7 Toward less infrastructure

The relay server is the main remaining dependency. A peer-to-peer mesh, with WireGuard directly between Pulsars, reduces it. The machinery already exists in the codebase:

- Firmware dual-session architecture (relay session plus P2P session).
- Conductor per-peer `boringtun` sessions with hole-punching and a 120 s rekey.
- LAN candidate discovery through TCP responses.
- Opportunistic direct handshake when endpoints are discovered.

A direct link also changes what the relay can see: it forwards ciphertext it cannot decrypt, whereas in the default star topology it can read traffic in transit.

**Current status: partially working.** The relay is becoming a bootstrap service: it introduces peers during provisioning and then steps out of the critical path. Locally, peers can already reach each other without it. What remains is formal peer discovery with no server involved (§4).

If that is completed, the infrastructure requirement could fall from "a relay you trust" to "a small computer in the same building" and, with pre-provisioned keys, to nothing but the devices themselves. The goal is for the system to behave like ordinary plumbing: something that works in the background.

### 3.8 The name

A pulsar is a rapidly rotating neutron star whose radio pulses arrive at very regular intervals. The most stable millisecond pulsars rival atomic clocks, and astronomers use them as timing references. The name points at the system's shared sense of time. The firmware synchronizes over SNTP, keeps the offset in a `critical_section::Mutex<Cell<u64>>`, and generates TAI64N timestamps for WireGuard replay protection. That gives every provisioned node a common time reference accurate to roughly ±50 ms.

Authentication rests on cryptographic keys rather than on passwords or trust in a vendor, which is why the guarantees described here are properties of the design rather than policies.

### 3.9 What it asks of you

The system asks you to understand how the link works, to manage your own keys, to run your own relay, to flash your own firmware, and to accept the responsibility that comes with that. The Covenant states the exit plainly: if the tool stops serving the user's autonomy, the user is encouraged to disconnect.

In return, you get a system whose behavior you can inspect and whose dependencies you can name. Observe Pulsar is an attempt to show that a self-run alternative to licensed, cloud-dependent devices is practical to build.

---

## 4. Readiness

### In plain terms

This is where we say plainly what is real. The core of the Assembly works on the bench: devices are set up over Bluetooth, connect through encrypted tunnels, reach each other directly or through a relay, and run in isolated roles. On the bench it has run for days at a time without failing or leaking memory. The router, the cellular link and several robustness features are still being built. And nothing has been tested in the field yet. Field tests come next, and vineyards are intended to be first.

### 4.1 Readiness assessment

#### Bench-proven

Validated on the bench. Not yet validated in the field.

| Capability | Evidence |
|---|---|
| **Encrypted handshake and connection layer between devices** | Noise_IKpsk2 handshake (initiator and responder) in `#![no_std]` firmware (`wg.rs`) and in the Conductor (`boringtun`). 64-bit sliding-window anti-replay, proactive rekey (120 s), seamless key rotation. |
| **Both connection paths, switching automatically** | Firmware dual-session architecture: a persistent relay session plus an opportunistic P2P session. Conductor per-peer `boringtun` sessions with hole-punching. A direct handshake triggers when a LAN or WAN endpoint is discovered through the `LAN:` tag in a TCP response. Kinetic streaming uses P2P when available and falls back to the relay transparently. Tested with known peers; formal server-free discovery is still to do (below). |
| **Isolation between system parts** | Compile-time pin assignment through the `RoleHardware` enum: Observer (I²C), Actor (GPIO out), Flow (soft-PWM), Kinetic (step/dir), each locking its pins at boot. The firmware refuses commands that contradict a role contract. Kernel-level tenant isolation through `ipset` `hash:net,net` silos and a single `iptables` rule. |
| **Offline setup and provisioning of new devices** | BLE GATT provisioning: the Conductor generates the node's X25519 keypair on the phone, registers it with the registry, assembles the `ProvisioningPayload` (keys, endpoint, IP, WiFi credentials, role, safe-state configuration), writes it over BLE, and the device reboots into the mesh. No internet required. |

#### In build

| Capability | Current state | Remaining |
|---|---|---|
| **Router** (`pulsar-router`) | ESP32-S3 firmware with a hand-written PPP/HDLC framer, LCP/IPCP state machines, IPv4 NAT (TCP/UDP/ICMP tracking, port allocation, timeouts), WiFi AP (`192.168.8.1/24`), DHCP and an HTTP provisioning server. | Bench bring-up and integration testing; end-to-end validation with the Conductor and Pulsars. |
| **Cellular data as a backup internet path** | SIM7600A on UART2: PPP link establishment, LCP/IPCP negotiation, NAT for LAN clients. WAN switching by explicit Conductor command (`WAN:CELLULAR` / `WAN:WIFI` / `WAN:NONE`). There is deliberately no auto-failover. | Validation against a live carrier link. The local mesh does not depend on this path; it matters when the Conductor is outside the mesh. |

#### Still to do

Engineering work with defined scope.

| Capability | Current state | Work remaining |
|---|---|---|
| **A second, more powerful chip variant** | `pulsar-object-01` targets the ESP32-C3. A `vision_cam` feature exists for ESP32 but is untested. The router already targets the ESP32-S3. | ESP32-S3 validation for `pulsar-object-01`; a unified HAL abstraction; feature-flag consolidation; thermal and power validation on the S3. |
| **Reconnecting cleanly after a full power-off** | Tombstone boot model implemented: Phase 1 is 120 s of strict validation, then wipe on failure; Phase 2 writes tombstone `0x414C4956` and retries indefinitely. SNTP sync on boot. | Power-loss stress testing across brownout scenarios; NVS encryption and secure boot to prevent corruption; watchdog integration for brownout detection. |
| **Local device discovery with no server involved** | P2P mesh partially working: firmware dual-session, Conductor per-peer sessions, LAN candidate discovery through the `LAN:` tag. | A formal peer-discovery protocol (an mDNS/SSDP equivalent over the mesh); a mesh routing schema for role-to-role communication (for example Kinetic ↔ Observer); a pre-provisioned-key workflow for zero-relay deployments. |
| **Bridging older industrial equipment into the mesh** | Link role defined (`0x06`) with UART/I²C/SPI pin placeholders. The router has `bridge_tx` / `bridge_rx` pins and UART1 reserved for legacy bridging. | Link role firmware (protocol translation such as Modbus RTU and proprietary serial to mesh packets); a router bridge daemon; protocol translation configurations. |
| **Weatherproof enclosure for extreme temperatures** | Per-role 3D-printed enclosures designed (STL files in the repository). ESP32-C3 modules are rated −40 °C to +85 °C by the manufacturer; that rating has not been verified by our own testing. | IP67/IP69K enclosure variants; thermal management for sustained +85 °C; connector sealing (M12, cable glands); UV-resistant materials (ASA, PETG-CF). |

| Category | Count |
|---|---|
| Bench-proven | 4 core capabilities |
| In build | 2 |
| Still to do | 5, each with defined scope |

The remaining gaps are engineering execution with known solutions and allocated code locations. The larger unknowns are empirical rather than architectural, and they are listed next.

#### What is not yet proven

- **Field durability.** Weather, dust, condensation, vibration, years of unattended running. None of this has been tested outside the workshop.
- **Real-world radio.** Range and reliability over actual terrain, vegetation and buildings. The nodes use WiFi and Bluetooth; kilometre-scale links would need an additional radio that is not part of the Assembly today.
- **Scale.** Bench clusters are small. Behaviour at tens or hundreds of nodes is untested.
- **Relay confidentiality.** In the current star topology the relay can read traffic in transit. Only direct P2P sessions are end-to-end, and those are partially working. An end-to-end layer over the relayed path is not part of the system today.
- **Security.** The design is built on well-studied primitives, but the implementation has not had an independent review.
- **Power and longevity.** Real solar and battery budgets, brownout behaviour, and months-long stability.

These are the questions the first field deployments exist to answer.

### 4.2 First intended deployments

#### Vineyards: the intended first field deployment

*Design intent. Nothing here is deployed yet.*

Vineyards are the natural first test: spread-out plots, irregular power, real and unpredictable weather, and growers who are often wary of sending farm data to someone else's cloud. The conditions are harsh enough to teach us something and nowhere near as extreme as a polar station, which makes this a faster, lower-stakes place to find out what breaks. It is also where the first revenue is most likely to come from.

| Role | Intended function | Intended hardware |
|---|---|---|
| **Observer** | Temperature, humidity, atmospheric pressure (BME280) | ESP32-C3, solar + battery, sealed enclosure |
| **Actor** | Irrigation valve control (solenoid + relay) | ESP32-C3, 12 V latching valve, watchdog lease |
| **Actor** | Frost protection (heater or wind-machine trigger) | ESP32-C3, high-current relay, thermal cutoff |

**Intended topology.** Each vineyard block is one Conductor plus three to five Pulsars. The Conductor runs on a ruggedized Android tablet or a Raspberry Pi 4. The relay runs on a small VPS or a local Pi. The router sits at the gateway and provides the cellular backup.

**Intended operator experience.** The viticulturist provisions new nodes over BLE, with no cloud account and no subscription, and the system is meant to carry on through frost, power outages and cellular dead zones.

#### What the first field deployment must prove

| Claim | Evidence we will collect |
|---|---|
| The architecture holds | Months of continuous operation with no architectural changes |
| The mesh works | Direct-path operation and clean fallback during relay maintenance, with no data loss |
| Roles absorb variation | The same firmware serving different sensors and actuators |
| Safety contracts hold | Actor leases executing correctly in a real frost event |
| Operators can run it themselves | Non-embedded engineers provisioning, operating and maintaining nodes themselves |
| No cloud required | Operation through intermittent connectivity |
| The economics work | Roughly $8–15 per node plus a small relay, against $100+ for commercial equivalents with subscriptions |

#### Further intended application areas

Same hardware, different sensor and actuator payloads. The role system is meant to absorb the variation: Observer reads, Flow regulates, Actor switches.

| Area | Examples | Roles |
|---|---|---|
| **Small-scale agriculture** | Greenhouse climate and CO₂ ventilation; livestock water tank level and temperature; mushroom-room humidity and air exchange | Observer, Flow |
| **Homes and workshops** | Gate actuator and gate motor; garage climate; workshop dust collection; bench power and fume extraction; CNC pendant; off-grid cabin solar monitoring, battery disconnect and water pump | Observer, Actor, Flow, Kinetic |
| **Small industry and infrastructure** | Water-district tank level and pump control replacing SCADA radio; remote telecom site battery and generator monitoring; agricultural cold storage | Observer, Actor |

For the legacy-radio cases, the driver is to replace clear-text radio with encrypted, auditable, kernel-isolated links and no cloud dependency. That depends on the Link role, which is still a stub.

#### The common thread

| Factor | What it is designed to look like |
|---|---|
| **Provisioning** | The operator stands at the device, opens the app, scans, names it, done. The design goal is about 30 seconds per device. |
| **Operation** | The app shows a role-appropriate interface: a gauge for Observer, a toggle for Actor, a slider for Flow, a joystick for Kinetic. |
| **Failure** | Power cut: tombstone boot, then infinite retry (stress testing still to do). Relay down: the mesh routes locally. Cellular dead: the operator switches the router's WAN to WiFi; there is no automatic failover. |
| **Maintenance** | `CLRMEM` wipes a device to virgin. A new phone provisions a new Conductor, and the truth downloads from the Librarian. |
| **Cost** | Expected $8–15 per node (ESP32 module plus sensor or actuator) and a small VPS. No per-node fees. |

### 4.3 The first product boundary

**What the Assembly consists of today**

- **Firmware:** Lab, Observer, Actor, Flow, Kinetic roles. Vision, Link and Intent are stubs.
- **Router:** in build. Cellular and WiFi WAN, NAT, PPP, HTTP provisioning.
- **Server:** Conductor, Secondary and Pulsar CRUD; `ipset` silos; a backup loop.
- **App:** BLE provisioning, mesh routing, role-based interface, telemetry graphs.

**What an operator is meant to receive:** a box of nodes, an app, a relay address, and their own keys. No account creation. No terms of service. No data leaving their infrastructure.

**What the builder provides:** source code, schematics, STL files, documentation. No recurring revenue. No lock-in. No telemetry.

This is not yet a finished product, and we won't pretend otherwise. It is a working core, proven on the bench, deliberately minimal, and ready for field testing.

---

## 5. Where the Architecture Goes

### In plain terms

The design assumes difficult conditions: little power, patchy or no connectivity, no one on site to fix things. Those conditions are common outside cities and offices. That is why a system designed for a vineyard is also a candidate for a polar station, a ship or a mountain slope. This section describes where the design is aimed, what has to happen before anyone should trust it there, and the order we intend to do it in.

### 5.1 Why these environments

The architecture was designed around constraint: limited power, intermittent connectivity, untrusted networks, physical isolation. Those are not edge cases. They are the ordinary operating conditions in many remote and industrial settings.

The same properties that make the system a reasonable fit for a vineyard also make it a candidate for a polar station, a ship, or a remote mine.

### 5.2 Environments it is designed for

*These describe how the architecture is meant to meet each environment. Capabilities such as store-and-forward and satellite bursting are design direction, not shipped features. These are environments the design is aimed at, not engagements: no outreach to organizations in these fields has happened yet.*

| Environment | Constraints | How the architecture is meant to answer |
|---|---|---|
| **Polar research** | −60 °C, months of darkness, katabatic winds, no infrastructure, satellite-only comms priced per megabyte | Mesh-first operation with the relay optional; deep sleep and solar-friendly duty cycles; Actor lease contracts so a heater cannot run away; Conductor-side store-and-forward with burst transmission over satellite. At these temperatures the electronics are outside their rated range, so heated, insulated enclosures are part of the job (§4.1). |
| **Maritime & offshore** | Salt spray, vibration, constant motion, satellite-only backhaul, classification-society rules | Conformal coating and sealed enclosures; an on-board mesh; store-and-forward over satellite; a type-approval path to pursue only if maritime proves worthwhile. No jurisdictional cloud dependency in international waters. |
| **Exploration & remote field science** | Desert, jungle, altitude, caves, glaciers; weeks from support; every gram counted | A small deployment intended to fit in one transport case and be provisioned the night before with no internet; solar power; operation by a single person. |
| **Industrial & critical infrastructure** | Brownfield sites, legacy equipment, regulation, uptime | The Link role to bridge legacy serial equipment (still a stub); kernel isolation in place of VLANs; an encrypted mesh in place of clear-text radio; the Librarian in place of a SCADA historian. Relevant to regimes such as NERC CIP, though the Assembly is not certified for any of them. |

The core design stays the same across these environments. What changes is the enclosure, the power system and the radio setup.

### 5.3 Two longer-term possibilities

**Space (speculative).** Some of the properties that matter at a polar station would also matter on a lunar surface or in a deep-space relay: store-and-forward behaviour, a common time reference (TAI64N), local operation without ground contact, and cryptographic identity without key upload. The obstacle is the silicon. ESP32-C3 and S3 chips are not radiation-hardened, so this would need a rad-hard microcontroller running the same firmware, and WireGuard carried over delay-tolerant networking instead of UDP. The software design could carry over; the hardware is the gate.

**Defence.** Observe Pulsar is infrastructure, not a weapons system. The properties that serve scientists in hostile places (no cloud dependency, no vendor in the supply chain, no metadata trail, keys loaded in a controlled facility before deployment) are also of interest to anyone who operates where communications are contested. We treat that as a possible future, not an active one.

### 5.4 Assurance path

Nothing below has been done. These are the steps we intend to pursue, in roughly this order, before asking anyone to trust the system with real consequences.

| Step | What it is | Status |
|---|---|---|
| **Field validation** | Deploy real hardware into the first environments and see what breaks | Planned: vineyards first (§4.2) |
| **Independent security review** | An outside review, including a red-team engagement, before anything is trusted with real deployments | To pursue |
| **IEC 62443-4-2, Security Level 2** | A component-level industrial security standard. The hardening items in §1.4 (secure boot, flash encryption, signed firmware, SBOM, CI gates) are the engineering path toward it. | To pursue; not certified |
| **Maritime type approval** | IEC 60945 and classification-society approval | Only if maritime proves worthwhile |
| **Regime-specific alignment** | NERC CIP, NIST 800-171 and similar, as deployments demand | Later |

### 5.5 Roadmap

| Phase | Focus |
|---|---|
| **Now** | Finish the in-build and still-to-do lists (§4.1): bring up the router and cellular path, validate the ESP32-S3 variant, complete power-loss recovery, formalize server-free discovery, build the Link role and legacy bridging, and engineer a real enclosure. |
| **Next** | Field validation in the first intended environments, then an independent security review of what the field exposes. |
| **Later** | The IEC 62443 evidence package; self-organizing mesh at scale (100+ nodes) and autonomous role negotiation through the Intent role; harder environments such as maritime and polar, as enclosures and certification allow; the space and defence paths if they prove real. |

---

## 6. Conclusion

Observe Pulsar starts from a simple design choice: no cloud broker, no subscription, and no vendor between you and your own devices. The rest of the Assembly follows from it. The firmware speaks WireGuard natively. Device identity is cryptographic and portable. Setup happens over Bluetooth with no internet. Tenants are separated by the kernel rather than by policy. Each device's role is fixed in its firmware. And with the router in build, the path from a device to the phone in your hand can be run end to end without third-party services, apart from the carrier SIM if you choose cellular.

Today, that is proven on the bench and nowhere else. We have said so plainly throughout, because claims about ownership and independence are only worth making if they are accurate. The field record, the independent review and any certifications are all still ahead, and we intend to document each of those steps as it happens.

The next steps are straightforward: finish the router, close the remaining engineering gaps, put real nodes into a real vineyard and see what breaks, and have the result reviewed by people whose job is to find faults. After that, harder environments can be considered.

The project is self-funded and intends to remain so. It is not a startup seeking investors or an exit. The aim is a modest, working piece of infrastructure that keeps existing because it is useful, and because its owners can keep running it themselves.

It is built for people who want to run their own systems and are willing to learn how they work. If that describes you, we would like to hear from you.

*Observe Pulsar: infrastructure you run yourself, with a clear account of what works today and what does not yet.*

---

**Liturgy Bureau** · liturgy.one · chris@liturgy.one
Observe Pulsar Overview · Version 3.0 (public) · rev. 2026-10-03



