# Observe Pulsar — Sovereign IoT Ecosystem

**Observe Pulsar** is a vertically integrated IoT platform where every device communicates exclusively through encrypted WireGuard tunnels — no cloud brokers, no MQTT, no third-party accounts.

---

## The Vision: Trust-Minimized Networking

Observe Pulsar is not an IoT platform in the commercial sense. It is a sovereign networking stack — a complete, vertically-integrated system for creating private, encrypted, kernel-isolated peer-to-peer links between a mobile controller and ESP32 microcontrollers. 

- **Zero Cloud Intermediary**: No subscription layers, no data extraction, and no external dependency on any corporation's uptime.
- **Math as the Authority**: The system does not ask for your trust; it makes trust unnecessary by enforcing security at the mathematical and kernel levels.
- **Vertical Rigor**: One person. Full stack. Silicon to enclosure to app to server.

---

## Technical Architecture

The ecosystem is built on a three-phase lifecycle designed for maximum security and user autonomy.

### 1. BLE Provisioning (Physical Contact)
Cryptographic identity is bestowed, not self-generated. The mobile "Conductor" app generates X25519 keypairs and transmits them via Bluetooth Low Energy directly to the hardware. Keys never leave the user's possession.

### 2. Boot Dispatch (Role Materialization)
On reboot, the firmware connects to WiFi and synchronizes time via SNTP. A role dispatcher materializes the device's specific logic (Observer, Actor, Flow, etc.) based on the signed provisioning payload.

### 3. The Live Tunnel (WireGuard Runtime)
The Conductor app and Firmware establish a direct WireGuard tunnel. All commands and telemetry travel through this encrypted link. The VPS relay is "dumb"—it only forwards encrypted packets and enforces kernel-level firewall isolation between tenants.

---

## Ecosystem Assessment Summary

*Conducted 2026-04-15*

### Technical Maturity
- **Protocol Design**: Coherent and lightweight, utilizing a single binary protocol (Postcard) end-to-end.
- **Firmware**: Genuinely excellent bare-metal WireGuard implementation running on Embassy async Rust (ESP32-C3).
- **Security**: Kernel-level silo allocation ensures each user operates inside their own cryptographically sealed network segment.

### Philosophical Coherence
The project follows a strict **Covenant**:
- **Zero-Extraction**: No data is harvested or stored.
- **Non-Transactional**: The system is a tool, not a service.
- **User Autonomy**: If the tool ceases to serve the user's freedom, it is designed to be disconnected or bypassed.

### Sociological Positioning
Observe Pulsar is built for a specific audience: technically sophisticated builders, privacy-motivated individuals, and creative technologists who demand absolute control over their physical and digital environments.

---

## The Stack

| Layer | Technology |
|-------|------------|
| **Firmware** | Rust (`#![no_std]`), ESP32-C3, Embassy async runtime |
| **Networking** | Custom bare-metal WireGuard (Noise_IKpsk2) |
| **Mobile App** | Angular + Tauri (Android/iOS), BoringTun, smoltcp |
| **Relay Server** | Rust (Axum), SQLite, Linux kernel `ipset` isolation |
| **Protocol** | Shared Postcard binary serialization crate |
| **Hardware** | Custom 3D-printed enclosures for specific device roles |

---

## General Verdict

**Observe Pulsar is a rare kind of artifact: a system that is technically serious, philosophically coherent, and culturally intentional.**

It is not a consumer product. It is a systems-integration achievement that proves high-performance, sovereign IoT is possible without compromising on security or autonomy. 

*The Pulsar pulses. Whether it builds a network around it depends entirely on whether the right person hears the signal.*
