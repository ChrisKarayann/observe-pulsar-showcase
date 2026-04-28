# Observe Pulsar — Public Description

---

## One-Liner

*For bios, social headers, forum signatures:*

**Observe Pulsar** — Sovereign IoT devices connected over encrypted WireGuard tunnels. No cloud. No subscription. Full Rust stack, from bare-metal firmware to mobile app.

---

## Short Pitch

*For forum posts, video descriptions, README headers:*

Observe Pulsar is a vertically integrated IoT platform where every device communicates exclusively through encrypted WireGuard tunnels — no cloud brokers, no MQTT, no third-party accounts.

Devices are provisioned over Bluetooth from a mobile app, receive their cryptographic identity on first contact, and join an isolated private network segment. Each tenant's devices are firewalled from every other tenant at the kernel level. If the central relay disappears, the cryptographic keys remain with the user — not a corporation.

The entire stack is written in Rust: bare-metal firmware (`#![no_std]`), the VPS relay server, and the cross-platform mobile app (Android/iOS). Hardware enclosures are custom-designed and 3D printed.

One person. Full vertical: silicon to enclosure to app to server.

---

## Extended Pitch

*For CrowdSupply, blog post, detailed video narration:*

Every commercial IoT device you buy today routes your commands through someone else's server. Your light switch talks to Amazon. Your thermostat reports to Google. Your security camera streams to a data center you've never seen. If that company shuts down, changes their terms, or gets breached — your devices become bricks or liabilities.

**Observe Pulsar eliminates the middleman entirely.**

Each Pulsar device is an ESP32-based module that establishes a point-to-point encrypted WireGuard tunnel to a lightweight relay server. The relay cannot read the traffic — it only forwards encrypted packets. The cryptographic keys are generated on your phone and never leave your possession. Devices are provisioned over Bluetooth Low Energy in seconds: scan, name, provision, done.

The network architecture enforces tenant isolation at the Linux kernel level. If ten users share the same relay, none of them can see, reach, or interfere with another's devices. Each user operates inside their own cryptographically sealed network segment.

Devices are assigned functional roles at provisioning time — sensor, actuator, relay, controller — and execute their role-specific logic autonomously on the microcontroller. Commands and telemetry travel through the encrypted tunnel with no intermediary parsing, translating, or storing your data.

### The Stack

| Layer | Technology |
|-------|------------|
| **Firmware** | Rust, `#![no_std]`, running on ESP32 with Embassy async runtime |
| **Networking** | Custom WireGuard implementation on bare metal — not a wrapper, not a library binding |
| **Mobile App** | Angular + Tauri, compiling natively for Android and iOS |
| **Relay Server** | Rust (Axum), managing peer registration and kernel-level firewall rules |
| **Protocol** | Shared binary serialization crate used identically across firmware, app, and server |
| **Hardware** | Custom 3D-printed enclosures designed for each device role |

This is not a platform that asks for your trust. It is a system that makes trust unnecessary — because the math is doing the work.
