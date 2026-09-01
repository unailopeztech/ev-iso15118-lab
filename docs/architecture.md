# EVerest Architecture

## System Overview

EVerest is a modular open-source EV charging software stack maintained by the Linux Foundation Energy. Each functional component runs as an independent module. Modules communicate exclusively via MQTT — no module calls another module directly.

CSMS (backend)
      |
      | OCPP 2.0.1 (WebSocket)
      |
   OCPP201 (C++)
      |
      | SessionEvents
      |
   EvseManager (C++)
      |
      | ISO15118_charger interface
      |
   IsoMux (C++)
      |
      |-- si negocia ISO 15118-2 --> EvseV2G (Josev Python)
      |
      |-- si negocia ISO 15118-20 -> Evse15118D20 (Josev Python)
      |
      | SDP (IPv6 UDP multicast / fe80::)
      |
   EV simulado (PyEvJosev)
      |
      | SLAC (PLC sobre cable CCS)
      |
   CCS Cable

## Module Communication

Modules communicate via MQTT using a publish/subscribe pattern. The interface YAML files define the contract between modules — what commands a module accepts and what variables it publishes.

Example: when Josev completes CableCheck, it publishes `cable_check_finished` via EVEREST_CTX. The EvseManager receives this via the `ISO15118_charger` interface and sets `hlc_allow_close_contactor = true`.

## Key Modules

| Module | Language | Role |
|--------|----------|------|
| `IsoMux` | C++ | Arbitrates between ISO 15118-2 and ISO 15118-20 |
| `EvseV2G` | C++ + Python (Josev) | ISO 15118-2 SECC implementation |
| `Evse15118D20` | C++ + Python (Josev) | ISO 15118-20 SECC implementation |
| `PyEvJosev` | Python | EV-side simulation (EVCC) |
| `EvseManager` | C++ | Coordinates hardware, auth, and protocol layers |
| `SlacSimulator` | C++ | Simulates PLC signaling (SLAC) |
| `DCSupplySimulator` | C++ | Simulates DC power supply |
| `IMDSimulator` | C++ | Simulates isolation monitoring device |
| `OCPP201` | C++ | OCPP 2.0.1 charge point implementation |
| `Auth` | C++ | Token validation and authorization |
| `EnergyManager` | C++ | Smart charging and energy optimization |

## Interface Contract

Every module declares its interface in a YAML file under `interfaces/`. Example from `ISO15118_charger.yaml`:

- **cmds** — commands other modules can call on this module
- **vars** — variables this module publishes for others to consume

The `evse_manager` module connects to the ISO 15118 module via this interface, receiving variables like `dc_ev_target_voltage_current` and `current_demand_started`.

## Configuration

The entire system is configured via a single YAML file that declares which modules are active and how they connect. Example: `config-sil-dc-isomux.yaml` defines the SIL simulation with ISO 15118 IsoMux.

Each module entry specifies:
- Which module implementation to use
- Module-specific configuration parameters
- Connections to other modules via their interface IDs