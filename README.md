# dspic33ak-timer-hal

Small, readable Timer HAL for Microchip dsPIC33AK devices.

This repository is intended to contain reusable timer services extracted from
the dsPIC33AK evaluation projects. The goal is not to hide everything behind a
framework, but to provide a compact timer layer that is easy to read, test,
modify, and adapt.

## Status

Initial public repository bootstrap.

The first Timer1 1 ms tick implementation has been validated in
`dspic33ak-hal-starter` before creating this standalone HAL repository.

Current validation baseline:

* Device: dsPIC33AK512MPS512
* Compiler: XC-DSC v3.31.01
* DFP: Microchip dsPIC33AK-MP DFP 1.3.185 or compatible
* Validation project: `dspic33ak-hal-starter`
* Validation branch: `hal-timer-integration`

## Design policy

This HAL is intentionally small.

* Timer ownership is explicit.
* Board/application code owns clock bring-up.
* Board/application code owns pin/PPS routing when relevant.
* HAL-owned interrupt vectors must be documented clearly.
* No RTOS dependency is required.
* No dynamic memory allocation is used.

## Initial scope

Planned first import:

* Timer1-based 1 ms tick service
* Monotonic millisecond counter
* Simple blocking timeout helper usage by consumer projects
* Documentation for Timer1 ownership and limitations

Out of scope for the first public import:

* Full timer peripheral abstraction
* PWM / output compare / input capture
* RTOS scheduler integration
* Timer2 high-resolution profiling unless separately validated
* Production-certified timing framework

## Repository layout

```text
src/
  Timer HAL source files
docs/
  Design notes and migration notes
```

## Relationship to starter projects

The Timer HAL is developed and hardware-validated in starter/evaluation projects
first, then moved here once the behavior is small, readable, and reusable.

Known validation source:

* `dspic33ak-hal-starter` branch `hal-timer-integration`

## License

MIT No Attribution License (MIT-0). See `LICENSE`.

Attribution is appreciated but not required.
