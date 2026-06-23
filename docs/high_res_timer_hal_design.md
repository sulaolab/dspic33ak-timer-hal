# dsPIC33AK High-Resolution Timer HAL Design

## Purpose

This HAL provides a small Timer2-based free-running counter for short interval
measurement and lightweight profiling on dsPIC33AK projects.

The implementation was developed and hardware-validated in
`dspic33ak-hal-starter` on branch `hal-timer-integration`, then moved here for
standalone reuse alongside the Timer1 tick HAL.

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

- Timer2 high-resolution counter initialized successfully.
- Raw count progression was observed at boot.
- 100 MHz conversion sanity checks passed.

Additional consumer validation:

- Repository: `sulaolab/perseus_512_96K`
- Branch: `main`
- Commit: `5201e7887f5dda33942076359a75042b66e4f7fc`
- Audio/TDM load-monitor reporting continued to run through the public API.

Functional integration was validated. Absolute Timer2 frequency accuracy has not
yet been independently characterized with oscilloscope or logic-analyzer
measurement.

## Public API

```c
dspic33ak_high_res_timer_status_t dspic33ak_high_res_timer_init(
    const dspic33ak_high_res_timer_config_t *config);

dspic33ak_high_res_timer_status_t dspic33ak_high_res_timer_deinit(void);

bool dspic33ak_high_res_timer_is_present(void);

bool dspic33ak_high_res_timer_is_initialized(void);

uint32_t dspic33ak_high_res_timer_get_count(void);

uint32_t dspic33ak_high_res_timer_elapsed_count(uint32_t start_count);

uint32_t dspic33ak_high_res_timer_count_to_us(uint32_t count);

uint32_t dspic33ak_high_res_timer_count_to_us_x10(uint32_t count);

uint32_t dspic33ak_high_res_timer_elapsed_us(uint32_t start_count);

uint32_t dspic33ak_high_res_timer_elapsed_us_x10(uint32_t start_count);
```

## Config

```c
typedef struct {
    uint32_t timer_clk_hz;
    bool run_in_idle;
} dspic33ak_high_res_timer_config_t;
```

`timer_clk_hz` is the actual clock seen by Timer2. The HAL does not know or
configure PLLs, CLKGENs, or board clock routing.

`run_in_idle` maps to `T2CONbits.SIDL`: `false` stops Timer2 in CPU Idle,
`true` lets Timer2 continue in Idle.

## Timer Ownership

Timer2 is reserved by this HAL after successful init.

The HAL configures:

- `T2CON`
- `TMR2`
- `PR2`
- `_T2IF`
- `_T2IE`

Other modules must not use Timer2 unless ownership is deliberately changed.

The initial standalone repository targets devices with the validated Timer2 and
interrupt-symbol layout. Other dsPIC33AK devices may require verification of
`T2CON` / `TMR2` / `PR2`, `_T2IF`, and `_T2IE`.

## Interrupt Ownership

This HAL does not define `_T2Interrupt()` and does not enable Timer2 interrupts.

Timer2 is configured as an interrupt-free free-running counter. The application
does not need to provide a Timer2 vector for this HAL.

## Counter Configuration

The high-resolution timer uses Timer2 as a 32-bit free-running counter:

```text
clock source = internal peripheral clock
prescaler = 1:1
TCKPS = 0b00
PR2 = 0xFFFFFFFF
interrupt = disabled
```

At 100 MHz Timer2 input:

```text
1 count = 10 ns
full 32-bit cycle = about 42.95 seconds
```

Elapsed-count calculations use unsigned 32-bit subtraction and are
wraparound-safe when the measured interval is shorter than one full Timer2
counter cycle.

## Error Handling

`dspic33ak_high_res_timer_init()` rejects:

- `config == NULL`
- `timer_clk_hz == 0`
- devices where the required Timer2 SFRs or interrupt symbols are not present
  at compile time

`dspic33ak_high_res_timer_deinit()` returns
`DSPIC33AK_HIGH_RES_TIMER_ERR_NOT_INITIALIZED` if called before successful init.

## Getter Semantics

Before successful init and after deinit:

- `dspic33ak_high_res_timer_get_count()` returns 0.
- `dspic33ak_high_res_timer_elapsed_count()` returns 0.
- conversion helpers return 0 because the stored Timer2 clock is 0.

After successful init:

- `dspic33ak_high_res_timer_get_count()` returns the raw Timer2 count.
- `dspic33ak_high_res_timer_elapsed_count(start)` returns `TMR2 - start`.
- conversion helpers truncate integer results and saturate to `UINT32_MAX` if
  the converted value does not fit in `uint32_t`.

The `*_us_x10()` helpers return 0.1 us units. For example, a return value of
1234 means 123.4 us.

## ISR Usage Guidance

The conversion helpers use 64-bit division. Interrupt code should normally store
raw counts only, then convert counts in foreground/status code.

Recommended ISR pattern:

```c
uint32_t start_count = dspic33ak_high_res_timer_get_count();

/* measured ISR work */

uint32_t elapsed_count =
    dspic33ak_high_res_timer_elapsed_count(start_count);
```

The foreground/status path can later call:

```c
uint32_t elapsed_us_x10 =
    dspic33ak_high_res_timer_count_to_us_x10(elapsed_count);
```

## Non-Goals

- Timer2 periodic interrupts.
- Capture, compare, PWM, gated timer, or external counter modes.
- Busy-wait delay API.
- Generic timer instance abstraction.
- Cycle-accurate tracing across a full 32-bit wrap or longer interval.
