# EVerest Interfaces

## What is an Interface?

An interface in EVerest is a YAML contract that defines how modules communicate. Every module declares what commands it accepts (`cmds`) and what variables it publishes (`vars`). No module calls another module directly — all communication goes through MQTT using these contracts.

Interface files are located in `everest-core/interfaces/`.

## Structure

```yaml
cmds:
  command_name:
    description: What this command does
    arguments:
      argument_name:
        type: string/boolean/object/number
vars:
  variable_name:
    description: What this variable contains
    type: string/boolean/object/number
```

## ISO15118_charger Interface

The most important interface for EV charging — defines the contract between the ISO 15118 module (Josev) and the EvseManager.

### Key Commands (EvseManager → ISO 15118)

| Command | Description |
|---------|-------------|
| `setup` | Configure EVSE ID and debug mode at startup |
| `session_setup` | Send payment options at each session start |
| `set_powersupply_capabilities` | Set min/max voltage, current, power limits |
| `update_dc_present_values` | Update current voltage and current from power supply |
| `dlink_ready` | Signal that SLAC completed successfully |
| `cable_check_finished` | Notify that isolation check passed |
| `stop_charging` | Stop the charging process |

### Key Variables (ISO 15118 → EvseManager)

| Variable | Description |
|----------|-------------|
| `dc_ev_target_voltage_current` | Target voltage and current requested by the EV |
| `dc_ev_maximum_limits` | Maximum current, voltage, power the EV can accept |
| `current_demand_started` | Signals that DCChargeLoop has started |
| `current_demand_finished` | Signals that DCChargeLoop has ended |
| `evcc_id` | EV MAC address — unique vehicle identifier |
| `dlink_terminate` | Request to terminate the data link |
| `dlink_pause` | Request power saving mode (session pause) |
| `start_cable_check` | Trigger isolation monitoring |
| `start_pre_charge` | Trigger pre-charge phase |

## Connection to Code

When Josev publishes via EVEREST_CTX:
```python
EVEREST_CTX.publish('DC_EVTargetVoltageCurrent', ev_targetvalues)
```

This corresponds to the `dc_ev_target_voltage_current` variable in `ISO15118_charger.yaml`. The EvseManager receives it and forwards the target values to the DC power supply module.

## The Double Safety Check

The EvseManager only allows closing the DC contactor when two independent conditions are met:

```cpp
if (shared_context.hlc_allow_close_contactor    // ISO 15118 says OK
    and shared_context.iec_allow_close_contactor) // IEC 61851 PWM says OK
```

- `hlc_allow_close_contactor` — set when ISO 15118 CableCheck passes
- `iec_allow_close_contactor` — set when IEC 61851 PWM pilot signal is correct

Both must be true before high voltage DC is applied.