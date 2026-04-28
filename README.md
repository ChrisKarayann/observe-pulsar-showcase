# Observe Pulsar — Sovereign IoT Ecosystem

![Observe Pulsar Showcase](pulsarface.png)

**Observe Pulsar** is a vertically integrated IoT platform where every device communicates exclusively through encrypted WireGuard tunnels — no cloud brokers, no MQTT, no third-party accounts. Full Rust stack, from bare-metal firmware to mobile app.


---

## 00. Preface: The Sovereignty Doctrine

Observe Pulsar is not an IoT platform in the commercial sense. It is a sovereign networking stack — a complete, vertically-integrated system for creating private, encrypted, kernel-isolated peer-to-peer links between a mobile controller and ESP32 microcontrollers. 

There is no cloud intermediary, no subscription layer, no data extraction, and no external dependency on any corporation's uptime. This is not a platform that asks for your trust. It is a system that makes trust unnecessary — because the math is doing the work.

---

## 01. Technical Assessment

### The Architecture — Three Phases

1.  **Phase 1 — BLE Provisioning (Physical Contact).** Cryptographic identity is bestowed, not self-generated. The mobile "Conductor" app generates X25519 keypairs and transmits them via Bluetooth Low Energy directly to the hardware. Key generation happens on the trusted device — the phone — not the microcontroller.
2.  **Phase 2 — Boot Dispatch (Role Materialization).** On reboot, the firmware connects to WiFi and synchronizes time via SNTP. A role dispatcher reads the provisioning payload and materializes the device's specific logic (Observer, Actor, Flow, Kinetic, Vision, Link, Intent) based on a compiled role dispatcher.
3.  **Phase 3 — Runtime (The Live Tunnel).** The Conductor app and Firmware establish a direct WireGuard tunnel. The VPS relay is "dumb"—it only forwards encrypted packets and enforces kernel-level firewall isolation between tenants. The VPS never sees plaintext traffic.

### Bare-Metal WireGuard (Noise_IKpsk2)
The ecosystem implements the complete **Noise_IKpsk2** protocol from scratch in `no_std` Rust. This includes X25519 Diffie-Hellman, ChaCha20-Poly1305 AEAD, BLAKE2s hashing, and TAI64N timestamps for replay protection. This is not a library wrapper; it is a careful systems-level implementation running on an ESP32-C3 with no OS.

### Maturity by Domain
| Domain | Status |
|---|---|
| **Protocol Design** | Clean, no bloat, coherent end-to-end. |
| **Firmware** | Production-grade bare-metal WireGuard + Embassy async. |
| **Registry Server** | Functional registry with kernel-level `ipset` isolation. |
| **Conductor App** | Cross-platform (Tauri/Angular) with integrated BoringTun stack. |
| **System Integration** | Coherent 4-layer vertical stack. |

---

## 02. Philosophical Assessment

### The Covenant
The project is governed by a "Covenant"—a statement of design axioms that are architecturally enforced:
- **Zero-Extraction Rule**: The system is incapable of harvesting user data.
- **Non-Transactional Constraint**: The link is a tool for autonomy, not a service for consumption.
- **Tender Cut-off**: Architectural enforcement of privacy via mathematics rather than policy.

### Epistemic Honesty
The system acknowledges the "Principal Tension": the requirement for a VPS relay. While the relay is technically "dumb" and trust-minimized, it remains a central coordination point. This is an honest intermediate position on the path toward full decentralization.

---

## 03. Sociological Assessment

### The Audience
Observe Pulsar selects for its own users. It targets technically sophisticated individuals—builders, creative technologists, and privacy-motivated operators—who are willing to study the mechanics of the link in exchange for absolute sovereignty.

### Adoption Barriers
The onboarding path requires technical competence (VPS management, firmware flashing, cryptographic stewardship). This is not a defect; it is a structural filter. We do not provide "easy" answers for those unwilling to understand the tools they use.

---

## 04. General Verdict

**Observe Pulsar is a rare kind of artifact: a system that is technically serious, philosophically coherent, and culturally intentional.**

It is not a trivial system. The bare-metal WireGuard implementation alone is a significant technical achievement. The coherent Postcard protocol across three runtime environments (Embedded, Server, Mobile) represents a genuine systems integration milestone.

*The Pulsar pulses. Whether it builds a network around it depends entirely on whether the right person hears the signal.*

---

## 05. Request Access

The source code for the full ecosystem is currently hosted in a **Private Repository** to protect the architectural integrity during this phase of development.

If you have read this assessment and wish to audit the source, contribute to the stack, or deploy a Pulsar network:

1.  Prepare your **Public SSH Key**.
2.  Contact the architect (Chris Karayannidis) with a brief description of your use case.
3.  Upon approval, you will be provided with a **Deploy Key** or repository access.

> [!NOTE]
> *Observe Pulsar is a labor of technical intensity. Access is granted to those who share the vision of sovereign, decentralized infrastructure.*
