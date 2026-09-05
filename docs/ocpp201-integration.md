# OCPP 2.0.1 Integration in EVerest

## Overview

The `OCPP201` module (`everest-core/modules/EVSE/OCPP201/OCPP201.cpp`) connects the charging session managed by ISO 15118 and EvseManager with the CSMS (Charge Point Management System) via OCPP 2.0.1 over WebSocket.

## Position in the Stack
CSMS (backend)
|
| OCPP 2.0.1 (WebSocket / TLS)
|
OCPP201 (C++)
|
| subscribes to SessionEvents
|
EvseManager (C++)
|
| ISO15118_charger interface
|
ISO 15118 / Josev

## SessionEvent Flow

The OCPP201 module subscribes to `SessionEvents` published by EvseManager and converts them into OCPP 2.0.1 messages sent to the CSMS:

| SessionEvent | OCPP 2.0.1 Message |
|-------------|-------------------|
| `TransactionStarted` | `TransactionEvent (Started)` |
| `ChargingStarted` | `TransactionEvent (Updated)` |
| `ChargingPausedEV` | `TransactionEvent (Updated)` |
| `ChargingPausedEVSE` | `TransactionEvent (Updated)` |
| `TransactionFinished` | `TransactionEvent (Ended)` |
| `SessionStarted` | `StatusNotification` |
| `SessionFinished` | `StatusNotification` |

## ISO 15118 Integration Points

OCPP 2.0.1 and ISO 15118 are connected at three points:

### 1. Plug & Charge Configuration
The CSMS enables or disables Plug & Charge by setting a variable via OCPP:

```cpp
const auto iso15118_pnc_enabled_response = this->charge_point->request_value<bool>(
    ControllerComponents::ISO15118Ctrlr,
    Variable{PNC_ENABLED_VAR_NAME}
);
```

The `ISO15118Ctrlr` component in OCPP 2.0.1 controls ISO 15118 behavior from the CSMS.

### 2. Certificate Management
During Plug & Charge, the EV requests a V2G certificate. The charger forwards this to the CSMS via OCPP:

```cpp
auto ocpp_response = this->charge_point->on_get_15118_ev_certificate_request(
    conversions::to_ocpp_get_15118_certificate_request(certificate_request)
);
```

OCPP message: `Get15118EVCertificate` — the CSMS acts as a bridge to the V2G PKI (Public Key Infrastructure).

### 3. Charging Needs Notification
Parameters negotiated during ISO 15118 `ChargeParameterDiscovery` are reported to the CSMS:

```cpp
// NotifyEVChargingNeedsRequest
```

OCPP message: `NotifyEVChargingNeeds` — the CSMS can use this to optimize the charging profile.

## Key Design Principle

The OCPP201 module knows nothing about ISO 15118 internals. It only sees:
- SessionEvents from EvseManager
- Meter values from the power meter
- Authorization results from the Auth module

ISO 15118 and OCPP 2.0.1 are decoupled — the EvseManager acts as the bridge between the two worlds.

## Personal Context

At Veltium Smart Chargers, OCPP 2.0.1 was implemented from scratch on ESP32 using ESP-IDF and FreeRTOS. The messages handled included `BootNotification`, `Heartbeat`, `StartTransaction`, `MeterValues`, and `StatusNotification` — the same messages that EVerest's OCPP201 module sends to the CSMS.

The key difference: in EVerest, OCPP sits on top of a full ISO 15118 stack. In the Veltium implementation, OCPP communicated directly with the hardware without ISO 15118.