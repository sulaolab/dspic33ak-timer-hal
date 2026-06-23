# dspic33ak-timer-hal

> Want to run it on hardware first?
> Start with [dspic33ak-hal-starter](https://github.com/sulaolab/dspic33ak-hal-starter),
> which vendors validated snapshots of the dsPIC33AK HAL repositories and
> provides a ready-to-build MPLAB X project for the dsPIC33AK Curiosity board.

Small, readable Timer HAL for Microchip dsPIC33AK devices with the validated
Timer1, Timer2, and interrupt-register layout.

This repository provides:

- a Timer1-based 1 ms monotonic tick HAL for timeout and heartbeat sources
- a Timer2-based 32-bit free-running high-resolution counter for profiling and
  short interval measurement

The goal is not to create a full timer framework, but to provide compact,
explicit, easy-to-review timer building blocks for evaluation, FAE demos, and
early software architecture experiments.

## Status

Current validation target:

- Device: dsPIC33AK512MPS512
- Compiler: XC-DSC v3.31.01
- DFP: Microchip dsPIC33AK-MP DFP 1.3.185 or compatible
- Validation project: `dspic33ak-hal-starter`
- Validation branch: `hal-timer-integration`
- Validation commit: `8feb1e82ca61073a6d81397e2c9095e3ea83fafe`
- Standalone compile check:
  - `src/dspic33ak_tick_timer.c` compiles for `33AK512MPS512` and
    `33AK128MC106`
  - `src/dspic33ak_high_res_timer.c` compiles for `33AK512MPS512` and
    `33AK128MC106`

Integration regression checks in `dspic33ak-hal-starter`:

- 1 Hz heartbeat continued to run
- UART startup log continued to run
- I2C scan and I2C loopback continued to run
- CAN internal loopback / live timeout path continued to run
- RGB LED / switch demo continued to run
- Timer1 vector was application-owned and forwarded to the HAL handler
- Timer2 high-resolution counter initialized, advanced, and passed a 100 MHz
  conversion sanity check

Functional integration was validated. Absolute tick accuracy has not yet been
independently characterized with oscilloscope or logic-analyzer measurement.

## Supported Devices

Hardware validated:

- dsPIC33AK512MPS512

Register-level compatibility reviewed and compile checked:

- dsPIC33AK128MC106

`dsPIC33AK128MC106` is expected to be compatible by Timer1/Timer2 register
symmetry, but should be listed as hardware-validated only after a final HAL build
is tested on that target.

Other dsPIC33AK devices may require verification of the Timer1, Timer2, and
interrupt-symbol layout, especially `T1CON` / `TMR1` / `PR1`, `_T1IF`,
`_T1IE`, `_T1IP`, `T2CON` / `TMR2` / `PR2`, `_T2IF`, and `_T2IE`.

## Design Policy

This HAL is intentionally small.

- Timer clock frequencies are supplied by the caller.
- Clock tree and PLL setup are outside this HAL.
- The application owns interrupt vectors.
- The HAL owns Timer1 registers after successful tick timer init.
- The HAL owns Timer2 registers after successful high-resolution timer init.
- No busy-wait delay API is provided.
- No RTOS or scheduler dependency exists.
- No dynamic memory allocation is used.
- No XC-DSC / DFP bitfield types are exposed in the public API.

## Scope

In scope:

- Timer1-based 1 ms monotonic tick
- Timer2-based 32-bit free-running high-resolution counter
- Timeout callback source for other HALs and application code
- Short interval measurement and profiling with raw Timer2 counts
- Configurable Timer1 input clock
- Configurable Timer2 input clock
- Configurable Timer1 interrupt priority
- Init / deinit
- Presence and initialized-state queries
- Application-owned interrupt forwarding for Timer1

Out of scope:

- RTOS software timers
- Busy-wait delay functions
- Capture / compare / PWM
- External pulse counting
- Gated timer mode
- Callback scheduler
- Generic multi-instance timer abstraction

## Files

```text
src/
  dspic33ak_tick_timer.c
  dspic33ak_tick_timer.h
  dspic33ak_high_res_timer.c
  dspic33ak_high_res_timer.h
docs/
  tick_timer_hal_design.md
  high_res_timer_hal_design.md
```

## Integration

1. Add the needed `src/*.c` files to the project.
2. Add `src/` to the compiler include path.
3. Include the required public header.
4. Supply the actual timer input clock in `timer_clk_hz`.
5. Define `_T1Interrupt()` in the application and forward it to
   `dspic33ak_tick_timer_irq_handler()` when using the tick timer.

## Basic Usage

The board/application code brings up the clock tree, then passes the actual
Timer1 input clock to the tick HAL:

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

The high-resolution counter is initialized separately:

```c
#include <xc.h>

#include "dspic33ak_high_res_timer.h"

const dspic33ak_high_res_timer_config_t high_res_timer_config = {
    .timer_clk_hz = 100000000u,
    .run_in_idle = false,
};

if (dspic33ak_high_res_timer_init(&high_res_timer_config) !=
    DSPIC33AK_HIGH_RES_TIMER_OK) {
    while (1) {
        Nop();
    }
}
```

For ISR profiling, store raw counts in the ISR and convert later:

```c
uint32_t start_count = dspic33ak_high_res_timer_get_count();

/* measured work */

uint32_t elapsed_count =
    dspic33ak_high_res_timer_elapsed_count(start_count);
uint32_t elapsed_us_x10 =
    dspic33ak_high_res_timer_count_to_us_x10(elapsed_count);
```

## API Summary

Tick timer:

- `dspic33ak_tick_timer_init()` - validate config, select prescaler, configure
  Timer1, clear the counter, enable the interrupt, and start Timer1.
- `dspic33ak_tick_timer_deinit()` - disable the interrupt, stop Timer1, clear
  Timer1 registers, clear the millisecond counter, and mark the HAL
  uninitialized.
- `dspic33ak_tick_timer_is_present()` - return whether the required Timer1 SFRs
  and interrupt symbols are present at compile time.
- `dspic33ak_tick_timer_is_initialized()` - return the internal initialized
  state.
- `dspic33ak_tick_timer_get_ms()` - return the current millisecond counter, or
  0 before successful init and after deinit.
- `dspic33ak_tick_timer_irq_handler()` - clear the Timer1 interrupt flag and
  increment the millisecond counter.

High-resolution timer:

- `dspic33ak_high_res_timer_init()` - configure Timer2 as a 32-bit free-running
  counter and start it.
- `dspic33ak_high_res_timer_deinit()` - stop Timer2, clear Timer2 registers, and
  mark the HAL uninitialized.
- `dspic33ak_high_res_timer_is_present()` - return whether the required Timer2
  SFRs and interrupt symbols are present at compile time.
- `dspic33ak_high_res_timer_is_initialized()` - return the internal initialized
  state.
- `dspic33ak_high_res_timer_get_count()` - return the raw Timer2 count, or 0
  before successful init and after deinit.
- `dspic33ak_high_res_timer_elapsed_count()` - return wraparound-safe raw count
  elapsed from a saved start count.
- `dspic33ak_high_res_timer_count_to_us()` - convert raw counts to integer
  microseconds.
- `dspic33ak_high_res_timer_count_to_us_x10()` - convert raw counts to 0.1 us
  units.

## Timer Ownership

Timer1 is reserved by the tick HAL after successful init.

The HAL does not define `_T1Interrupt()`. Consumer projects must provide the
interrupt vector and call `dspic33ak_tick_timer_irq_handler()` from it.

Timer2 is reserved by the high-resolution timer HAL after successful init. This
HAL does not enable Timer2 interrupts and does not define `_T2Interrupt()`.

## Timer1 Prescaler and Period Calculation

The tick HAL targets a fixed 1 kHz interrupt rate so that each counter increment
represents approximately 1 ms. The period is exact when the supplied Timer1
clock is evenly divisible by 1000.

Prescaler candidates are checked in this order:

```text
1:1, 1:8, 1:64, 1:256
```

The first candidate whose rounded count fits in the 32-bit `PR1` range is used:

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

## Timer2 High-Resolution Counter

The high-resolution HAL configures Timer2 as an interrupt-free 32-bit
free-running counter:

```text
TCKPS = 0b00
PR2 = 0xFFFFFFFF
```

At 100 MHz Timer2 input, one count is 10 ns and a full 32-bit cycle is about
42.95 seconds.

Conversion helpers use 64-bit division. Interrupt code should normally store raw
counts and convert them later in foreground/status code.

## Known Follow-Up

Before declaring the standalone repository production-ready, define the tick
error policy for input clocks that cannot generate an exact 1 ms period. The
current hal-starter validation uses 100 MHz Timer1 input, where the generated
period is exact.

## Notes

- This repository does not include Microchip DFP header files.
- This HAL is the timer layer only; clock setup belongs to the board/application.
- CMSIS or RTOS timer wrappers, if needed later, should live in separate layers.

## License

MIT No Attribution License (MIT-0). See [LICENSE](LICENSE).

Attribution is appreciated but not required.
