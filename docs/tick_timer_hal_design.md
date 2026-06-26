# dsPIC33AK Tick Timer HAL Design

## Purpose

This HAL provides a small Timer1-based 1 ms monotonic tick service for dsPIC33AK
projects. It is intentionally narrower than a generic timer framework: the first
public scope is a readable timeout/heartbeat tick source.

The implementation was developed and hardware-validated in
`dspic33ak-hal-starter` on branch `hal-timer-integration`, then moved here for
standalone reuse.

## Source Baseline

Validated source project:

- Repository: `sulaolab/dspic33ak-hal-starter`
- Branch: `hal-timer-integration`
- Commit: `8feb1e82ca61073a6d81397e2c9095e3ea83fafe`
- Validated device: `dsPIC33AK512MPS512`
- Toolchain: XC-DSC v3.31.01
- DFP: Microchip dsPIC33AK-MP DFP 1.3.185 or compatible

Standalone compile checks:

- `33AK512MPS512` with Microchip dsPIC33AK-MP DFP 1.3.185
- `33AK128MC106` with Microchip dsPIC33AK-MC DFP 1.2.125

Integration regression checks:

- UART startup log continued to run.
- 1 Hz heartbeat continued to run.
- I2C scan and I2C loopback continued to run.
- CAN internal loopback / live timeout path continued to run.
- RGB LED and switch demo continued to run.
- The application-owned Timer1 vector forwarded to the HAL handler.

Functional integration was validated. Absolute tick accuracy has not yet been
independently characterized with oscilloscope or logic-analyzer measurement.

## Public API

```c
#define DSPIC33AK_TICK_TIMER_DEFAULT_IRQ_PRIORITY   4u

dspic33ak_tick_timer_status_t dspic33ak_tick_timer_init(
    const dspic33ak_tick_timer_config_t *config);

dspic33ak_tick_timer_status_t dspic33ak_tick_timer_deinit(void);

bool dspic33ak_tick_timer_is_present(void);

bool dspic33ak_tick_timer_is_initialized(void);

uint32_t dspic33ak_tick_timer_get_ms(void);

void dspic33ak_tick_timer_irq_handler(void);
```

No legacy `systick_*` compatibility wrappers are provided. `SysTick` is a
Cortex-M-specific term and is avoided for dsPIC33AK Timer1 code.

## Config

```c
typedef struct {
    uint32_t timer_clk_hz;
    uint8_t irq_priority;
    bool run_in_idle;
} dspic33ak_tick_timer_config_t;
```

`timer_clk_hz` is the actual clock seen by Timer1. The HAL does not know or
configure PLLs, CLKGENs, or board clock routing.

`irq_priority` must be 1 through 7. Priority 0 and values above 7 are rejected.
`DSPIC33AK_TICK_TIMER_DEFAULT_IRQ_PRIORITY` provides the recommended convenience
value used by the starter integration; applications can supply another valid
priority when needed.

`run_in_idle` maps to `T1CONbits.SIDL`: `false` stops Timer1 in CPU Idle,
`true` lets Timer1 continue in Idle.

## Timer Ownership

Timer1 is reserved by this HAL after successful init.

The HAL configures:

- `T1CON`
- `TMR1`
- `PR1`
- `_T1IF`
- `_T1IE`
- `_T1IP`

Other modules must not use Timer1 unless ownership is deliberately changed.

The initial standalone repository targets devices with the validated Timer1 and
interrupt-symbol layout. Other dsPIC33AK devices may require verification of
`T1CON` / `TMR1` / `PR1`, `_T1IF`, `_T1IE`, and `_T1IP`.

## Interrupt Ownership

The HAL does not define `_T1Interrupt()`.

The application owns the vector and forwards it:

```c
void __attribute__((interrupt, context)) _T1Interrupt(void)
{
    dspic33ak_tick_timer_irq_handler();
}
```

This matches the policy used by the other standalone HAL repositories: the HAL
provides an ISR-processing hook, while the application/project owns the vector.

## Prescaler and Period Calculation

The tick rate is fixed at 1000 Hz. This keeps
`dspic33ak_tick_timer_get_ms()` semantically simple and suitable for timeout
callbacks.

Prescaler candidates:

```text
1:1, 1:8, 1:64, 1:256
```

The HAL checks candidates from smallest to largest and uses the first candidate
whose rounded count fits in the 32-bit `PR1` range:

```text
counts = round(timer_clk_hz / (prescaler * 1000))
PR1 = counts - 1
```

At 100 MHz Timer1 input, the selected configuration is exact:

```text
prescaler = 1:1
TCKPS = 0b00
PR1 = 99999
```

## Error Handling

`dspic33ak_tick_timer_init()` rejects:

- `config == NULL`
- `timer_clk_hz == 0`
- `irq_priority == 0`
- `irq_priority > 7`
- clocks whose calculated period cannot fit in 32-bit `PR1`
- devices where the required Timer1 SFRs or interrupt symbols are not present
  at compile time

If init fails, Timer1 registers are not modified by this HAL's configuration
sequence.

`dspic33ak_tick_timer_deinit()` returns
`DSPIC33AK_TICK_TIMER_ERR_NOT_INITIALIZED` if called before successful init.

## Counter Semantics

The internal counter is a volatile `uint32_t` millisecond counter. The
initialized flag is also volatile because it is written by normal code and read
from the interrupt handler. The counter is cleared on successful init and
increments in `dspic33ak_tick_timer_irq_handler()`.

Before successful init and after deinit, `dspic33ak_tick_timer_get_ms()` returns
0. `dspic33ak_tick_timer_deinit()` stops Timer1, clears the Timer1 registers,
clears the millisecond counter, and marks the HAL uninitialized.

Users should compare times with unsigned subtraction:

```c
if ((uint32_t)(now - previous) >= interval_ms) {
    previous = now;
}
```

On XC-DSC v3.31.01 with `-O2` for dsPIC33AK512MPS512,
`dspic33ak_tick_timer_get_ms()` was checked as generated assembly and reads the
counter with a single `mov.l`.

## Known Follow-Up

Before declaring the standalone repository production-ready, define and implement
a policy for input clocks that cannot generate an exact 1 ms tick. Options
include:

- accept nearest rounded period and document the error,
- reject configurations above a chosen ppm/error threshold,
- expose a helper for users to query the selected period/error.

The current hardware validation uses a 100 MHz Timer1 input, where the selected
period is exact.

## Non-Goals

- Generic timer instance abstraction.
- Capture, compare, PWM, gated timer, or external counter modes.
- Busy-wait delay API.
- RTOS software timers or schedulers.
