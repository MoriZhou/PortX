# PortX Specification Language (PSL) v0.1

## Purpose

PSL defines a standardized language for describing embedded software systems.

## Design Principles

- Human Readable
- AI Readable
- Vendor Independent
- Traceable
- Verifiable

## Requirement DSL

```yaml
System:
  CoolingFan

Requirements:
  - Temperature >= 60:
      FanSpeed: 100
```

## Module DSL

```yaml
Module:
  FanController

Inputs:
  - Temperature

Outputs:
  - FanSpeed
```

## Interface DSL

```yaml
Interface:
  GPIO

Functions:
  - Set
  - Clear
  - Read
```

## State Machine DSL

```yaml
StateMachine:
  Fan

States:
  - OFF
  - LOW
  - HIGH

Transitions:
  - OFF -> LOW
  - LOW -> HIGH
```

## Architecture DSL

```yaml
Layers:
  - Application
  - Service
  - HAL
  - BSP
```

## Constraint DSL

```yaml
Forbidden:
  - VendorSDKInApplication
  - DirectBSPAccess
```

## Future Extensions

- AUTOSAR Models
- Safety Rules
- Timing Constraints
- Memory Constraints
- Multi-Core Definitions
