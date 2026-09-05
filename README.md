# EV Charging Protocol Lab — ISO 15118 & V2G

Personal lab environment for studying ISO 15118, V2G, and the EVerest open-source stack.

## Motivation

ISO 15118 defines the communication protocol between electric vehicles and charging stations. Understanding this protocol stack — from the physical layer (SLAC) to the application layer (ISO 15118-2/20) and the backend (OCPP 2.0.1) — is essential for anyone working in EV charging infrastructure.

This repository documents my hands-on study of the full protocol stack using EVerest, the Linux Foundation open-source EV charging software suite.

## Stack Overview

    CSMS (backend)
            | OCPP 2.0.1 (WebSocket)
        EVSE (charger)
            | ISO 15118-2 / ISO 15118-20 (TCP/TLS over IPv6)
        EV (vehicle)
            | SLAC (PLC signaling over CCS cable)
        CCS Cable
        
## Documentation

Detailed technical documentation in the `docs/` folder:

- [Architecture Overview](docs/architecture.md) — EVerest module architecture and MQTT communication
- [Manager Lifecycle](docs/manager-lifecycle.md) — How the EVerest manager starts and supervises modules
- [Interfaces](docs/interfaces.md) — How modules communicate via YAML contracts
- [ISO 15118-2 DC State Machine](docs/iso15118-2.md) — Full DC charging sequence from SDP to SessionStop
- [ISO 15118-20](docs/iso15118-20.md) — V2G, ScheduleExchange Dynamic mode, and differences from ISO 15118-2
- [Charger State Machine](docs/charger-state-machine.md) — EvseManager C++ state machine and double safety check
- [OCPP 2.0.1 Integration](docs/ocpp201-integration.md) — How OCPP connects to ISO 15118 and the CSMS

## Environment

- **Host:** Windows 11 + WSL2 (Ubuntu 22.04)
- **Stack:** EVerest (everest-core) compiled from source
- **Simulation:** Software-in-the-Loop (SIL) with DC charging + ISO 15118 IsoMux
- **Tools:** Docker, Node-RED dashboard, MQTT Explorer, Mosquitto broker

## ISO 15118-2 DC State Machine

The SECC (Supply Equipment Communication Controller) implements the following state sequence for DC charging:

| State | Description |
|-------|-------------|
| SDP | Service Discovery Protocol — vehicle announces presence over IPv6 UDP multicast |
| SupportedAppProtocol | Negotiate protocol version (ISO 15118-2, ISO 15118-20, DIN 70121) |
| SessionSetup | Establish session ID, identify vehicle via EVCC ID (MAC address) |
| ChargeParameterDiscovery | Negotiate energy transfer mode and charging parameters |
| CableCheck | Verify cable isolation resistance via IMD — safety critical |
| PreCharge | Ramp EVSE output voltage to match EV battery voltage (ΔV < 20V per IEC 61851-23) |
| PowerDelivery | Open/close energy flow — used both to start and stop charging |
| CurrentDemand | Main charging loop — EV continuously requests target voltage and current |
| WeldingDetection | Verify DC contactor opened correctly after charging stops |
| SessionStop | Terminate or pause the session (pause preserves session ID for resumption) |

## ISO 15118-20 Key Differences

ISO 15118-20 introduces bidirectional power transfer (V2G) and smart charging:

- **AuthorizationSetup** — explicit negotiation of authorization method (EIM or Plug & Charge)
- **ServiceDiscovery / ServiceSelection** — extensible service model (DC, DC_BPT, AC, MCS)
- **ScheduleExchange** — critical for V2G: two control modes:
  - `SCHEDULED` — fixed charge/discharge schedule proposed by EVSE
  - `DYNAMIC` — real-time grid signals, vehicle responds cycle by cycle
- **DC_BPT service** — enables Vehicle-to-Grid energy flow in DCChargeLoop

## OCPP 2.0.1 Integration

EVerest connects ISO 15118 with the CSMS backend via OCPP 2.0.1:

- Plug & Charge enabled/disabled via `ISO15118Ctrlr` component variables
- ISO 15118 certificate requests forwarded to CSMS via `Get15118EVCertificate`
- EV charging needs (from `ChargeParameterDiscovery`) reported via `NotifyEVChargingNeeds`

## EVerest Architecture

EVerest uses a modular architecture where each component communicates via MQTT:

| Module | Role |
|--------|------|
| `IsoMux` | Arbitrates between ISO 15118-2 and ISO 15118-20 |
| `EvseV2G` | ISO 15118-2 SECC implementation (Josev) |
| `Evse15118D20` | ISO 15118-20 SECC implementation |
| `PyEvJosev` | EV-side simulation (EVCC) |
| `EvseManager` | Coordinates hardware, auth, and protocol layers |
| `SlacSimulator` | Simulates PLC signaling (SLAC) |
| `DCSupplySimulator` | Simulates DC power supply |
| `IMDSimulator` | Simulates isolation monitoring device |
| `OCPP201` | OCPP 2.0.1 charge point implementation |

## Results

## Results

Successfully compiled and ran EVerest SIL simulation with ISO 15118-20 DC IsoMux configuration on native Ubuntu 24.04:

![EVerest ISO 15118-20 DC Charging Session](EVerestUI.jpeg)

- SLAC matching completed
- EIM authorization flow completed  
- ISO 15118-20 full session completed:
  - SDP handshake over IPv6 link-local (fe80::)
  - SupportedAppProtocol negotiation — ISO 15118-20 DC selected
  - SessionSetup, AuthorizationSetup, Authorization
  - ServiceDiscovery, ServiceDetail, ServiceSelection — DC Dynamic mode
  - DcChargeParameterDiscovery — 300A/900V/150kW negotiated
  - ScheduleExchange — Dynamic control mode agreed
  - DcCableCheck — isolation verified at 900V
  - DCChargeLoop active — 22.1 kW charging confirmed
- Full MQTT message inspection via MQTT Explorer
- ISO 15118-2 and ISO 15118-20 state machines studied from Josev source code

**Note:** WSL2 blocks IPv6 link-local addresses required for SDP. Native Linux is required for a complete ISO 15118 session.

## Next Steps

- [ ] Dual boot Ubuntu — complete full ISO 15118-2 session
- [ ] Capture and analyze complete session logs
- [ ] Test V2G scenario with negative current (DC_BPT)
- [ ] Study ISO 15118-20 ScheduleExchange dynamic mode
- [x] Contribute to EVerest open source — [Issue #2688: ISO 15118 SDP fails on WSL2](https://github.com/EVerest/EVerest/issues/2688)

## References

- [EVerest](https://everest.github.io/)
- [Josev — Switch EV](https://github.com/SwitchEV/iso15118)
- [ISO 15118 — CharIN](https://www.charin.global/technology/iso-15118/)
- [OCPP 2.0.1 — OCA](https://www.openchargealliance.org/)
