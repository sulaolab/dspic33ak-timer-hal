# dspic33ak-timer-hal

Small, readable Timer HAL for Microchip dsPIC33AK devices with the validated
Timer1 and interrupt-register layout.

This repository provides a Timer1-based 1 ms monotonic tick HAL extracted from
`dspic33ak-hal-starter`. The goal is not to create a full timer framework, but
to provide a compact, explicit, easy-to-review tick source for evaluation,
FAE demos, and early software architecture experiments.

## Status

Current validation target:

* Device: dsPIC33AK512MPS512
* Compiler: XC-DSC v3.31.01
* DFP: Microchip dsPIC33AK-MP DFP 1.3.185 or compatible
* Validation project: `dspic33ak-hal-starter`
* Validation branch: `hal-timer-integration`
* Validation commit: `0823b5d2748c5bde1c466d5371b5b8bf4308b11d`
* Standalone compile check: `src/dspic33ak_tick_timer.c` compiles for
  `33AK512MPS512` with XC-DSC v3.31.01 / DFP 1.3.185.

Integration regression checks in `dspic33ak-hal-starter`:

* 1 Hz heartbeat continued to run
* UART startup log continued to run
* I2C scan and I2C loopback continued to run
* CAN internal loopback / live timeout path continued to run
* RGB LED / switch demo continued to run
* Timer1 vector was application-owned and forwarded to the HAL handler

Functional integration was validated. Absolute tick accuracy has not yet been
independently characterized with oscilloscope or logic-analyzer measurement.

## Supported Devices

Hardware validated:

* dsPIC33AK512MPS512

Register-level compatibility reviewed:

* dsPIC33AK128MC106

`dsPIC33AK128MC106` is expected to be compatible by Timer1 register symmetry,
but should be listed as hardware-validated only after a final HAL build is tested
on that target.

Other dsPIC33AK devices may require verification of the Timer1 and interrupt
register layout, especially `T1CON` / `TMR1` / `PR1`, `IFS1bits.T1IF`,
`IEC1bits.T1IE`, and `IPC6bits.T1IP`.

## Design Policy

This HAL is intentionally small.

* Timer clock frequency is supplied by the caller.
* Clock tree and PLL setup are outside this HAL.
* The application owns the Timer1 interrupt vector.
* The HAL owns Timer1 registers after successful init.
* No busy-wait delay API is provided.
* No RTOS or scheduler dependency exists.
* No dynamic memory allocation is used.
* No XC-DSC / DFP bitfield types are exposed in the public API.

## Scope

In scope:

* Timer1-based 1 ms monotonic tick
* Timeout callback source for other HALs and application code
* Configurable Timer1 input clock
* Configurable Timer1 interrupt priority
* Init / deinit
* Presence and initialized-state queries
* Application-owned interrupt forwarding

Out of scope:

* Timer2 high-resolution profiling
* RTOS software timers
* Busy-wait delay functions
* Capture / compare / PWM
* External pulse counting
* Gated timer mode
* Callback scheduler
* Generic multi-instance timer abstraction

## Files

```text
src/
  dspic33ak_tick_timer.c
  dspic33ak_tick_timer.h
docs/
  tick_timer_hal_design.md
```

## Integration

1. Add `src/dspic33ak_tick_timer.c` to the project.
2. Add `src/` to the compiler include path.
3. Include `dspic33ak_tick_timer.h`.
4. Supply the actual Timer1 input clock in `timer_clk_hz`.
5. Define `_T1Interrupt()` in the application and forward it to
   `dspic33ak_tick_timer_irq_handler()`.

## Basic Usage

The board/application code brings up the clock tree, then passes the actual
Timer1 input clock to the HAL:

```c
#include <xc.h>

#include "dspic33ak_tick_timer.h"

const dspic33ak_tick_timer_config_t tick_timer_config = {
    .timer_clk_hz = 100000000u,
    .irq_priority = 4u,
    .run_in_idle = false,
};

if (dspic33ak_tick_timer_init(&tick_timer_config) != DSPIC33AK_TICK_TIMER_OK) {
    while (1) {
        Nop();
    }
}
```

The application owns the Timer1 interrupt vector and forwards it to the HAL:

```c
void __attribute__((interrupt, context)) _T1Interrupt(void)
{
    dspic33ak_tick_timer_irq_handler();
}
```

Timeout users can pass the getter directly:

```c
.get_ms = dspic33ak_tick_timer_get_ms,
```

Use unsigned-difference timing for wraparound-safe intervals:

```c
uint32_t now = dspic33ak_tick_timer_get_ms();
if ((uint32_t)(now - previous) >= interval_ms) {
    previous = now;
}
```

## API Summary

* `dspic33ak_tick_timer_init()` - validate config, select prescaler, configure
  Timer1, clear the counter, enable the interrupt, and start Timer1.
* `dspic33ak_tick_timer_deinit()` - disable the interrupt, stop Timer1, clear
  Timer1 registers, and mark the HAL uninitialized.
* `dspic33ak_tick_timer_is_present()` - return whether Timer1 SFRs are present
  at compile time.
* `dspic33ak_tick_timer_is_initialized()` - return the internal initialized
  state.
* `dspic33ak_tick_timer_get_ms()` - return the current millisecond counter.
* `dspic33ak_tick_timer_irq_handler()` - clear the Timer1 interrupt flag and
  increment the millisecond counter.

## Timer Ownership

Timer1 is reserved by this HAL after successful init.

The HAL does not define `_T1Interrupt()`. Consumer projects must provide the
interrupt vector and call `dspic33ak_tick_timer_irq_handler()` from it.

## Prescaler and Period Calculation

The HAL targets a fixed 1 kHz interrupt rate so that each counter increment
represents approximately 1 ms. The period is exact when the supplied Timer1
clock is evenly divisible by 1000.

Prescaler candidates are checked in this order:

```text
1:1, 1:8, 1:64, 1:256
```

The first candidate whose rounded count fits in the 32-bit PR1 range is used:

```text
counts = round(timer_clk_hz / (prescaler * 1000))
PR1 = counts - 1
```

At 100 MHz Timer1 input:

```text
prescaler = 1:1
TCKPS = 0b00
PR1 = 99999
```

## Known Follow-Up

Before declaring the standalone repository production-ready, define the tick
error policy for input clocks that cannot generate an exact 1 ms period. The
current hal-starter validation uses 100 MHz Timer1 input, where the generated
period is exact.

## Notes

* This repository does not include Microchip DFP header files.
* This HAL is the timer layer only; clock setup belongs to the board/application.
* CMSIS or RTOS timer wrappers, if needed later, should live in separate layers.

## License

MIT No Attribution License (MIT-0). See [LICENSE](LICENSE).

Attribution is appreciated but not required.
