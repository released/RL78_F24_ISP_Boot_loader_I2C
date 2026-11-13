# RL78_F24_ISP_Boot_loader_I2C

I2C ISP Boot for R7F124FPJ5xFB. This device Boot supports only this update interface/protocol.

## Board and interface

| Item | Configuration |
| --- | --- |
| Clock | X1 4 MHz / CPU 40 MHz |
| Update link | IICA0 SCL / SDA P62 / P63, 7-bit address 0x20, 100 kHz |
| Diagnostic Log | UART0 TX / RX P15 / P16, 115200 8-N-1 |
| Update interrupt | IICA0 vector 0x28 |
| Timer vector | 0x2E |
| Flash implementation | RFD Type02; Flash-busy call chain executes from RAM |

Vector values are physical byte addresses, not channel numbers. UART0 carries text only. Cross UART TX/RX, or connect matching SCL/SDA with voltage-compatible pull-ups; share ground and verify logic levels.

## Image and interrupt ownership

Boot is 0x0000..0x1FFF (8 KiB), with at least 512 bytes spare. App vectors are 0x2000..0x20FF, metadata is 0x2100..0x21FF (256 bytes), and App code starts at 0x2200.
Maximum complete App BIN is 247 KiB; 0x3FC00..0x3FFFF is protected. Size includes vectors, metadata, padding and the four-byte CRC32.
Boot owns physical vectors: AA selects Boot handlers; 55 forwards through the App table. Normal handlers use RETI; BRK retains RETB.

## Configuration and application editing

- `image_metadata_config.h`: neutral MCU/Flash/format/Boot identity.
- `isp_transport_config.h`: selected ISP bus/protocol settings; metadata readers use the same values.
- `boot_app_handoff.h`: reserved RAM and AA/55 owner.
- `hal/isp/`: protocol/update engine and transport adapters; `hal/device/` and `hal/flash/` own hardware access.
- `hal/image/image_metadata.c/.h`, `hal/image/image_metadata_profile.c`: independent parser and expected profile.

App release/IRQ edits belong to the paired [App project](../RL78_F24_ISP_Boot_loader_I2C_app/README.md).

## Smart Configurator

Open this project's same-name `.scfg` from the CS+ Smart Configurator node.
After editing, use **Generate Code**, then **Rebuild** or the PowerShell command
below. Xmodem uses UART peripherals; UART0 remains the independent text Log.

Pre-build checks the MCU, pins, clocks, serial contract and actual generated
registers, then creates this project's `sc_peripheral_config.generated.h` for
runtime initialization. This header is an output, not an authoring input.
Update-link changes also need the paired Boot's `isp_transport_config.h` and
matching Boot/App/Transfer SC settings.

The existing runtime retains ISR, queue, mailbox, protocol and vector handling.
Configuration steps and the seven-bit I2C / SVA conversion are in the
[SC workflow](https://released.github.io/slide_boot_loader_for_RL78/serial-device.html#serial-scfg-workflow).

## Build and files

Run in this project root using installed CS+/CC-RL and Windows PowerShell:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .uild.ps1
```

Use `-Compiler <CubeSuite+.exe>` for a different installed CS+ path. Firmware authoring/build requires no Python or JSON editor. Generated INC/identity files are outputs, not inputs. App/Transfer wrappers build only themselves.

| File under DefaultBuild | Use |
| --- | --- |
| `boot0000_1FFF.hex` | Programmer installation of only the 8-KiB Boot region |

The full project HEX/MOT/ABS may contain IDE/device-file regions; it is not the Boot-only installation file.

## Tool operation and protocol

Install Boot with a programmer, then select **I2C > I2C ISP** in the Tool. Load the paired App update BIN and **Discover Target**. If App is running, use **Enter Boot**, then **Start Device Update** before the 30-second valid-App inactivity window expires. Discovery of Boot can proceed directly to update.

Framed ISP requests stage metadata and check START before ordered 64-byte writes. END checks complete length, stored trailer, full CRC32, metadata and App vectors before reset.

I2C sends one STOP-delimited request (maximum 110 bytes), then polls a fixed 111-byte read window: status 00=busy or A5=ready, followed by the framed reply and FF padding. Main-loop processing performs Flash work; the IICA ISR only moves mailbox bytes. ENTER/END reset waits for the complete ready read, final NACK and STOP.

Only a valid installed App supplies the version floor. Enabled policy permits same/newer versions. The explicit force operation skips version ordering for one transaction only; identity, layout, minimum Boot, metadata, vector and CRC checks remain active.

See [PROTOCOL.md](PROTOCOL.md) for byte layouts and transport behavior.
[Tool operation and wiring](https://released.github.io/slide_simple_renesas_isp_tool/#uart-device-operation) provides the common operation flow and report links.
Offline build/model results do not establish electrical, physical IRQ/Flash, dynamic-stack, restart or power-interruption acceptance.

## User code entry

`custom_func.c` provides `hardware_init()` for initialization and `loop()` for
foreground service. Add application initialization and foreground work through
these callbacks, retaining the existing communication service calls.
The selected main/startup adapter owns entry and handoff. Generated peripheral
files remain under `src/`; retain the project's MTPJ source selection when using
Code Generator or Smart Configurator, then rebuild to validate the image and vectors.

## Project layout and standalone build

This project includes its own source files, configuration headers and build tools.
A sibling Boot, App or Transfer checkout is not required to compile it.
Keep the project directory name aligned with its `.mtpj` filename; remove a ZIP download suffix when necessary.
Build with CS+ Rebuild, or run the local PowerShell entry:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\build.ps1
```

| Location | Responsibility |
| --- | --- |
| `custom_func.c` | Project initialization in `hardware_init()` and recurring work in `loop()`; runtime orchestration stays here |
| `hal/can` | CAN service, controller driver and CAN adapter, when used by this project |
| `hal/device` | Device, clock, peripheral and CPU handoff APIs |
| `hal/image` | Metadata parser/profile, image validation and software CRC |
| `hal/flash` | Flash access/geometry APIs, when required by the selected image |
| `hal/isp` | Update/Transfer protocol and transport processing APIs |
| Root configuration headers | Local MCU/Flash, handoff and communication contract |
| `tools` and root PowerShell helpers | Local metadata/IRQ generation and linked-image checks |
| `src` | Code Generator or Smart Configurator peripheral sources |

Only the sources registered in this project's CS+ configuration are linked.
Generated main/vector implementations must not become a second active entry.
Retain the `custom_func.c` callbacks and the selected startup adapter after code generation.

Boot and App each carry a local contract. Keep their MCU, Flash layout, image format,
IRQ ABI, handoff and communication settings compatible when installing a pair.
App releases are edited in `app_image_config.h` and `app_irq_handlers.h` in the App project.
Rebuild the selected App to regenerate its metadata, vector table, update BIN and timing BINs.
The App build does not read or rewrite a Boot checkout.

## I2C operation and companion projects

- [Custom Transfer](https://github.com/released/RL78_F24_I2C_Slave_Transfer)
- [ISP Boot](https://github.com/released/RL78_F24_ISP_Boot_loader_I2C)
- [ISP App](https://github.com/released/RL78_F24_ISP_Boot_loader_I2C_app)
- [Tool operation and measured App updates](https://released.github.io/slide_simple_renesas_isp_tool/serial-isp.html#f24-i2c-update-measurements)

## Repository updates

The project-local `github_push.bat -DryRun` previews this repository's source and
exact delivery. `github_push.bat` retains current-version board acceptance and
PUSH confirmation; it does not build, program a board, or create a repository.
Compiler outputs, IDE user settings, raw logs and engineering records stay local.
