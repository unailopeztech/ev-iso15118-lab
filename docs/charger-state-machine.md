# EvseManager Charger State Machine

## Overview

The `Charger` class in `everest-core/modules/EVSE/EvseManager/Charger.cpp` implements the high-level state machine of the charger. It sits above ISO 15118 and IEC 61851 — it coordinates both protocols and controls when the DC contactor closes and energy flows.

## States
Disabled
|
Idle
|
WaitingForAuthentication ←── waits for RFID token or PnC
|
PrepareCharging ←── SLAC + ISO 15118 negotiation
|
Charging ←── DCChargeLoop active (22.1 kW)
|
StoppingCharging ←── WeldingDetection + SessionStop
|
Finished

Additional states:
- `ChargingPausedEV` — EV requested pause
- `ChargingPausedEVSE` — charger requested pause
- `T_step_EF` — IEC 61851 error state
- `T_step_X1` — IEC 61851 transition state
- `SwitchPhases` — AC phase switching

## Key Transitions Seen in EVerest Logs
EVSE IEC Session Started: EVConnected
EVSE IEC Set PWM On (5.0%) → WaitingForAuthentication
EVSE IEC EIM Authorization received
EVSE IEC Transaction Started (0 kWh)
EVSE IEC DC mode. 5percent mode → PrepareCharging
EVSE ISO SLAC MATCHED
EVSE ISO D-LINK_READY (true)
EVSE ISO Start cable check...
EVSE IEC DC power supply set: 900V/2A → CableCheck
EVSE IEC DC power supply: switch ON → PreCharge
Ready to start charging → Charging

## The Double Safety Check

The most critical piece of logic in `Charger.cpp`. The charger only transitions from `PrepareCharging` to `Charging` when two independent conditions are both true:

```cpp
if (shared_context.hlc_allow_close_contactor
    and shared_context.iec_allow_close_contactor) {
    // close DC contactor, start energy flow
}
```

**`hlc_allow_close_contactor`** — set by ISO 15118 when:
- CableCheck passed (isolation resistance OK)
- PreCharge complete (voltage difference < 20V)

**`iec_allow_close_contactor`** — set by IEC 61851 when:
- PWM pilot signal is in the correct state
- EV signaled it is ready to charge

Both must be true simultaneously. This is a hardware safety requirement — closing the contactor with incorrect conditions would damage the contactor, the cable, or the battery.

## Connection to ISO 15118

The Charger state machine receives signals from ISO 15118 via the `ISO15118_charger` interface:
Josev (Python)
→ EVEREST_CTX.publish('cable_check_finished')
→ ISO15118_charger.yaml interface
→ Charger.cpp sets hlc_allow_close_contactor = true
→ PrepareCharging → Charging

## Connection to OCPP 2.0.1

The `OCPP201` module subscribes to `SessionEvents` from EvseManager:
Charger.cpp emits SessionEvent
→ TransactionStarted → OCPP TransactionEvent (Started) → CSMS
→ ChargingStarted → OCPP TransactionEvent (Updated) → CSMS
→ TransactionFinished → OCPP TransactionEvent (Ended) → CSMS

The CSMS receives real-time charging data via OCPP without knowing anything about ISO 15118 internals.