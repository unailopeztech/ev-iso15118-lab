# EVerest Manager Lifecycle

## What is the Manager?

The EVerest manager is the main process that spawns and supervises all module processes. It is the first process to start and the last to stop.

## States

| State | Description |
|-------|-------------|
| `Idle` | Manager alive but no modules running |
| `Initializing` | Reading configuration, setting up infrastructure |
| `StartingModules` | Spawning module processes, waiting for all to report ready |
| `Running` | All modules ready — system operational |
| `ShutdownRequested` | SIGINT/SIGTERM received, draining modules |
| `CrashShutdownInProgress` | A module exited unexpectedly, draining remaining modules |
| `RestartRequested` | Administrative restart requested, draining modules |
| `ForceTerminating` | Drain deadline exceeded, sending SIGTERM/SIGKILL |
| `ShutdownFinalizing` | All modules gone, deciding next action |
| `Exiting` | Final cleanup before process exit |

## Normal Startup Sequence

Idle → Initializing → StartingModules → Running

Seen in EVerest logs:

Manager state transition: Idle -> Initializing
Manager state transition: Initializing -> StartingModules
Starting 19 modules
All modules are initialized. EVerest up and running [2533ms]
Manager state transition: StartingModules -> Running

## Module Ready Mechanism

The manager subscribes to each module's ready topic on MQTT. Each module publishes ready when it finishes initialization. The manager transitions to `Running` only when every non-ignored module has published ready.

## Shutdown Sequence

Running → ShutdownRequested → ShutdownFinalizing → Exiting

With `--graceful-shutdown`: the manager publishes an MQTT shutdown signal so modules can run their shutdown handlers before being terminated.

Without `--graceful-shutdown` (default): modules are terminated immediately via SIGTERM, escalating to SIGKILL after a grace period.

## Crash Recovery

If a module exits unexpectedly:

Running → CrashShutdownInProgress → ShutdownFinalizing → Exiting (failure)

With `--recover-module-crashes`: the manager reloads the configuration and restarts all modules, up to a configurable cap.

## Key Design Principle

The manager never calls modules directly — all communication goes through MQTT. This means modules can crash and restart independently without affecting the manager process itself.