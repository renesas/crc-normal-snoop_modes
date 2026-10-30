# CRC (Normal, Snoop modes)

## Overview

This Software Building Block (SBB) demonstrates use of the Renesas CRC HAL together with a UART loopback data path. This document covers only:

- FPB-RA0L1
- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

The reference application calculates a CRC for four bytes of data, appends the calculated CRC to the transmitted data, transfers the data through a board-level UART loopback connection, validates the received data, and indicates the result using the terminal and on-board LED.

The implementation supports two CRC verification paths. Where CRC snoop is available, transmission is calculated in normal mode and reception is validated in snoop mode. Where snoop is unavailable, both transmitted and received data are checked using normal CRC calculation.

## Use case demonstrated by this SBB

The SBB demonstrates CRC-based data-integrity checking around a UART transfer:

1. Initialize the UART and CRC modules.
2. Calculate a CRC for four input bytes in normal mode.
3. Append the CRC to the input buffer.
4. Transfer the buffer over a physical UART TX/RX loopback.
5. Validate the received transfer using the CRC capability available on the target MCU.
6. Compare the transmitted and received buffers.
7. Indicate successful operation by blinking the on-board LED.
8. Display status and failure information on the serial terminal.

The supplied application starts a CRC operation when the user presses any key in the terminal.

## Behavior on the supported reference boards

The SBB adapts its CRC and UART path according to device capabilities.

| Board | CRC receive validation | UART loopback |
|---|---|---|
| FPB-RA0L1 | Normal mode; snoop is not supported | SAU UART channel 0 |
| FPB-RA2E3 | Uses the implementation selected by the device `BSP_FEATURE_CRC_HAS_SNOOP` capability | SCI UART channel 0 |
| FPB-RA4E2 | Normal mode; snoop is not supported | SCI UART channel 9 |
| FPB-RA6E2 | Normal mode; snoop is not supported | SCI UART channel 9 |
| FPB-RA8E1 | Uses the implementation selected by the device `BSP_FEATURE_CRC_HAS_SNOOP` capability | SCI UART channel 0 |

For boards where `BSP_FEATURE_CRC_HAS_SNOOP` is enabled, the received stream is checked using CRC snoop mode. Otherwise the received data has its CRC calculated in normal mode and that result is compared with the transmit-side CRC.

## Hardware setup

One Renesas RA board, a suitable USB cable, and one jumper wire are required by the supplied example documentation.

Connect the board USB debug port to the host PC. Then establish the UART loopback required for the selected board.

### FPB-RA0L1

- Place jumper **J9 on pins 1-2** to use the on-board debug functionality.
- Loop back SAU UART channel 0:
  - `SAU_UART0 TXD0 P101 (J2:11)` to `SAU_UART0 RXD0 P100 (J2:12)`.

### FPB-RA2E3

- Close **E25, E28, E46, and E47** to use J-Link OB VCOM for the serial terminal.
- Loop back SCI UART channel 0:
  - `SCI0 TXD0 P101 (J4:11)` to `SCI0 RXD0 P100 (J4:12)`.

### FPB-RA4E2

Loop back SCI UART channel 9:

- `SCI9 TXD9 P109 (J4:2)` to `SCI9 RXD9 P110 (J4:3)`.

### FPB-RA6E2

Loop back SCI UART channel 9:

- `SCI9 TXD9 P109 (J4:2)` to `SCI9 RXD9 P110 (J4:3)`.

### FPB-RA8E1

Loop back SCI UART channel 0:

- `SCI0 TXD0 P609 (J3:13)` to `SCI0 RXD0 P610 (J3:14)`.

## SBB core code

The reusable SBB functionality is split between the CRC and UART modules:

- `crc_ep.c`
- `crc_ep.h`
- `uart_ep.c`
- `uart_ep.h`

### CRC core

The CRC layer wraps the FSP CRC interface. It:

- Opens the configured CRC instance.
- Calculates a CRC using the configured polynomial and bit order.
- Enables CRC snoop mode when the target supports it.
- Retrieves the calculated snoop value when supported.
- Disables snoop mode when supported.
- Closes the CRC instance.

Snoop-specific APIs are compiled only when `BSP_FEATURE_CRC_HAS_SNOOP` is enabled.

### UART core

The UART layer provides a common interface over the UART implementation selected by BSP feature macros. It wraps:

- SCI_B UART when `BSP_PERIPHERAL_SCI_B_PRESENT` is defined.
- SCI UART when `BSP_PERIPHERAL_SCI_PRESENT` is defined.
- SAU UART otherwise.

It provides initialization, transmit, receive, cleanup, and callback handling. The UART callback sets completion flags when `UART_EVENT_RX_COMPLETE` and `UART_EVENT_TX_COMPLETE` occur.

## Reusable SBB code versus demo/reference application

### Reusable SBB code

The reusable layer consists of:

- `crc_ep.c` and `crc_ep.h` for CRC operations.
- `uart_ep.c` and `uart_ep.h` for the data-transfer abstraction used by this SBB.

These files isolate direct CRC and UART driver calls behind a smaller set of functions.

### Demo/reference application code

`crc_example_app.c` and `crc_example_app.h` provide the reference application. They are responsible for:

- Initializing the terminal console.
- Displaying example and FSP information.
- Starting a CRC operation after terminal input.
- Providing the example input bytes.
- Appending the calculated CRC to the transmit buffer.
- Performing the UART loopback sequence.
- Comparing transmitted and received data.
- Applying the normal-mode or snoop-mode validation flow.
- Driving the LED as a demonstration result indicator.
- Handling errors and entering a breakpoint trap.

The reference application's single entry point is:

```c
void crc_example_app_run(void);
```

The source identifies this function as the function called from `hal_entry.c`.

## Public functions to look at first

### CRC functions

#### `crc_init()`

```c
fsp_err_t crc_init(void);
```

Opens the CRC instance using `g_crc_ctrl` and `g_crc_cfg`.

#### `crc_calculate()`

```c
fsp_err_t crc_calculate(crc_input_t * const p_crc_input,
                        uint32_t * crc_value);
```

Calculates a CRC through the configured FSP CRC instance. This is the central CRC operation in both the transmit path and the normal-mode receive path.

#### `crc_snoop_enable()`

```c
fsp_err_t crc_snoop_enable(void);
```

Available only when `BSP_FEATURE_CRC_HAS_SNOOP` is enabled. It enables CRC snoop mode using the SBB `SEED_VALUE`.

#### `crc_calculated_value()`

```c
fsp_err_t crc_calculated_value(uint32_t * crc_value);
```

Available only when snoop is supported. It retrieves the CRC value calculated during snoop operation.

#### `crc_snoop_disable()`

```c
fsp_err_t crc_snoop_disable(void);
```

Disables snoop mode on targets that support it.

#### `crc_deinit()`

```c
void crc_deinit(void);
```

Closes the CRC instance if it is open.

### UART functions

#### `uart_init()`

```c
fsp_err_t uart_init(void);
```

Opens the generated UART instance. The source comments identify the configured baud rate as 115200.

#### `uart_write()` and `uart_read()`

```c
fsp_err_t uart_write(uint8_t const * const p_src, uint32_t const bytes);
fsp_err_t uart_read(uint8_t * const p_dest, uint32_t const bytes);
```

Start transmission and reception through the UART implementation selected by the BSP abstraction macros.

#### `uart_deinit()`

```c
void uart_deinit(void);
```

Closes the generated UART instance if it is open.

The UART implementation also contains `uart_cb()`, which records TX-complete and RX-complete callback events for the reference application's loopback operation.

## Reference configuration assumptions

The supplied implementation makes the following assumptions.

### CRC configuration

- The generated CRC instance is exposed as `g_crc_ctrl` and `g_crc_cfg`.
- The application processes **4 bytes** of source data (`NUM_BYTES`).
- The CRC seed used by the SBB is `0x00000000` (`SEED_VALUE`).
- CRC-8 produces a five-byte UART transfer: four source bytes plus one CRC byte.
- CRC-16 and CRC-CCITT produce a six-byte UART transfer: four source bytes plus two CRC bytes.
- The application supports its CRC-8, CRC-16, and CRC-CCITT handling paths, but reports other selected polynomial values as unsupported.
- The supplied README explicitly states that the application does **not** work with the CRC-32 polynomial.
- For FPB-RA0L1, the README states that only **CRC-CCITT** with **LSB** bit order is supported.

### UART configuration

- The generated UART instance is exposed through `g_uart_ctrl` and `g_uart_cfg`.
- The UART layer selects SCI_B, SCI, or SAU driver APIs at compile time using BSP peripheral feature macros.
- The loopback operation depends on UART transmit- and receive-completion callback events.
- The reference application uses `TIME_OUT = 10000000U` as a decrementing software timeout while waiting for both callbacks.

### LED configuration

The demo uses the first BSP-defined LED:

```c
g_bsp_leds.p_leds[RESET_VALUE]
```

Successful operation calls `toggle_led()`. Error handling sets the LED to `LED_ON` before deinitializing the CRC and UART modules and entering a breakpoint.

## What must change when adapting the SBB?

### UART peripheral and pins

The physical loopback is board-specific. If moving away from the supplied reference board configuration, the UART peripheral instance, channel, TX/RX pins, generated pin configuration, and physical signal routing need to match the new target hardware.

The supplied `uart_ep.h` already abstracts the driver family between SCI_B UART, SCI UART, and SAU UART through BSP feature macros, but it still expects compatible generated `g_uart_ctrl` and `g_uart_cfg` objects.

### CRC polynomial and bit order

The application uses `g_crc_cfg.polynomial` and `g_crc_cfg.bit_order` to determine how the CRC is handled. If these settings are changed in the CRC configurator, the selected configuration must remain compatible with the application's supported paths.

The README states that FPB-RA0L1 supports CRC-CCITT and LSB for this example, and that CRC-32 is unsupported by the application.

### CRC snoop capability

Do not assume snoop mode is available on every target. The source gates all snoop APIs and the snoop-based receive path with:

```c
BSP_FEATURE_CRC_HAS_SNOOP
```

FPB-RA0L1, FPB-RA4E2, and FPB-RA6E2 are explicitly identified by the supplied README as boards that operate without CRC snoop for this example.

### Clocks and baud rate

The UART source states that `uart_init()` opens a channel configured for 115200 baud, and the README specifies a 115200-baud terminal. The supplied files do not state the underlying peripheral clock settings needed to derive that baud rate on each reference board. Those settings must therefore be checked in the generated board/FSP configuration if clocks are changed.

### Timing

The application waits for UART TX and RX completion flags using a decrementing `TIME_OUT` loop and uses `R_BSP_SoftwareDelay()` with `TOGGLE_DELAY` when blinking the LED. These are implementation constants rather than measured timing guarantees.

### Memory and task priorities

The supplied files do not define an RTOS, task priorities, stack sizes, heap requirements, or overall SBB memory consumption. These requirements must be established by the integrating application if it introduces an RTOS or different execution model.

### Middleware and console

The demonstration's terminal interface depends on `console/uart_console.h` and standard I/O. This terminal interaction belongs to the reference application. An integration that uses the CRC/UART SBB functions without the interactive demo can provide its own application interface.

## Dependencies

### FSP CRC HAL

The CRC layer calls:

- `R_CRC_Open()`
- `R_CRC_Calculate()`
- `R_CRC_SnoopEnable()` when supported
- `R_CRC_CalculatedValueGet()` when supported
- `R_CRC_SnoopDisable()` when supported
- `R_CRC_Close()`

### FSP UART driver

Depending on BSP peripheral capabilities, the UART abstraction maps onto one of:

- `R_SCI_B_UART_*`
- `R_SCI_UART_*`
- `R_SAU_UART_*`

### FSP/BSP services

The application also uses:

- `R_FSP_VersionGet()`
- `R_IOPORT_PinWrite()`
- `R_BSP_SoftwareDelay()`
- Generated definitions exposed through `hal_data.h`
- BSP LED information through `g_bsp_leds`

### Console and C runtime

The reference application uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- Standard C functionality including `printf()`, `getchar()`, and `memcmp()`

### Hardware resources

The reference implementation requires:

- CRC peripheral
- A supported UART peripheral and channel
- TX and RX pins connected as a loopback
- An on-board BSP-supported LED for result indication
- USB/debug connection
- A serial terminal for the interactive reference application

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Port | J-Link On-Board port |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term, the README also specifies:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

## Demonstrated behavior and performance

### CRC data path

The demonstration starts with these four input bytes:

```text
05 02 03 04
```

A CRC is calculated for those four bytes and appended to the transfer buffer. The total UART transfer is five bytes for CRC-8 or six bytes for CRC-16/CRC-CCITT.

### Verification with snoop support

When `BSP_FEATURE_CRC_HAS_SNOOP` is enabled:

1. The transmit-side CRC is calculated in normal mode.
2. CRC snoop mode is enabled.
3. The buffer is transmitted and received through UART loopback.
4. The snoop CRC value is retrieved.
5. Snoop mode is disabled.
6. The operation is considered successful by the application when the snoop CRC value is zero and the transmitted and received buffers match.

### Verification without snoop support

When snoop is not available:

1. The transmit-side CRC is calculated in normal mode.
2. The buffer is transmitted and received through UART loopback.
3. A CRC is calculated for the received data in normal mode.
4. The application checks the transmit and receive CRC values and compares the transmitted and received buffers.

### Result indication

When the validation succeeds and the buffers match, the application prints a success message and blinks the on-board LED. Error handling prints an error, turns the LED on, deinitializes the CRC and UART modules, and enters a breakpoint.

### Performance measurements

The supplied source and README do **not** provide measured CRC throughput, UART throughput, CPU utilization, memory footprint, interrupt latency, execution time, or other benchmark results. `TIME_OUT` and `TOGGLE_DELAY` are implementation constants and should not be interpreted as performance measurements.

This SBB should therefore be treated as a functional CRC/UART integration reference rather than a performance benchmark.

## File ownership and modification guidance

| File | Classification | Guidance |
|---|---|---|
| `crc_ep.c` | Reusable SBB CRC implementation | Modify when intentionally changing the reusable CRC wrapper or supported CRC behavior. |
| `crc_ep.h` | Reusable SBB CRC interface | Modify when changing CRC-facing declarations or integration constants. |
| `uart_ep.c` | Reusable SBB UART implementation | Modify when changing the UART wrapper or callback behavior required by the SBB. |
| `uart_ep.h` | Reusable SBB UART interface/driver abstraction | Modify carefully when changing the SCI_B/SCI/SAU abstraction or UART-facing interface. |
| `crc_example_app.c` | Demo/reference application | Appropriate location for changes to demonstration sequencing, sample data, result presentation, or terminal-driven behavior. |
| `crc_example_app.h` | Demo/reference application interface | Contains the reference-app entry point and display strings. |
| `readme.txt` | Existing example documentation | Provides the reference hardware, loopback, CRC restriction, and terminal setup information used by this document. |

The source includes `hal_data.h` and depends on generated FSP objects such as `g_crc_cfg`, `g_crc_ctrl`, `g_uart_cfg`, and `g_uart_ctrl`. The supplied file set does not enumerate the associated generated files, so this document does not classify individual generated project files beyond those explicitly provided.

## Adopting a newer SBB release after local modifications

Changing `crc_ep.*` or `uart_ep.*` creates local differences in the reusable SBB layer. A later SBB release may change the same interfaces or implementation, requiring those local changes to be reconciled.

To reduce upgrade conflicts:

- Keep demonstration-specific behavior in `crc_example_app.*` where possible.
- Keep product-specific presentation, input handling, and test data out of the reusable CRC/UART wrappers unless they are genuinely part of the required SBB behavior.
- Keep changes to `crc_ep.*` and `uart_ep.*` focused and track why each change is required.
- Keep generated FSP configuration distinct from hand-maintained source changes.
- When adopting a newer SBB release, compare the reusable files with the locally modified versions and reapply only customizations that remain necessary.
- Revalidate the selected CRC polynomial and bit order, snoop capability, UART driver mapping, TX/RX configuration, callback completion behavior, physical loopback, and result checks after integration.

The supplied files do not define an automated source-migration or compatibility mechanism for future SBB versions, so compatibility after an upgrade must be validated by the integrating project.

## Recommended reading order

1. **`crc_ep.h`** to understand CRC constants and the reusable CRC API.
2. **`crc_ep.c`** to see how the CRC FSP driver is wrapped, including conditional snoop support.
3. **`uart_ep.h`** to understand the SCI_B/SCI/SAU compile-time UART abstraction.
4. **`uart_ep.c`** to see UART initialization, transfers, and callback completion flags.
5. **`crc_example_app.c`** to see the complete CRC calculation, UART loopback, validation, LED, and error-handling sequence.
6. **`crc_example_app.h`** to see the reference application entry point and generated presentation strings.
7. **`readme.txt`** for the original board-specific hardware connections and terminal setup.
