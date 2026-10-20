# ADF Peripheral Driver

This module provides an abstraction layer for the **audio digital filter** (`ADF`) of STM32 MCUs: the receiver of the bit-stream of
**one PDM digital microphone** (or a pair on one data line) with the CIC and the reshape filter, the high-pass filter and the gain,
producing the PCM audio samples, together with the **sound activity detector** that watches the signal level while the rest of the
MCU sleeps and wakes it when a sound (a voice) is detected. It offers the continuous acquisition with the DMA streaming, the
configuration of the detector and the clock generation for the microphones. It is a low-level MCAL module: it owns the ADF hardware, the
application and the other modules never touch the registers.

The public interface (`Adf_Port.h`, `Adf_Types.h`) is **identical for all supported families**. The implementation
for a family lives in its own branch (`Dev/<Family>` for development, `Releases/<Family>` for the released versions), so
a module or an application written against the interface does not change when the MCU family changes.

This README is the landing page and the **specification of the interface** (master). The family branches contain their
own README with everything that is specific to the family: the instances of the device, their settings and the status of
the implementation.

> **Status: design.** No implementation exists yet, the interface below is a proposal. Stories and branches follow.

---

## Intent and purpose

- The ADF is the **always-listening front end** of a voice or a sound activated product: the microphone and the filter run in a very
  low power state in the low-power mode, and the **sound activity detector (SAD)** compares the level of the audio (a smoothed value of the signal
  against a noise threshold) and raises the wake-up event when the sound appears. The application then switches to the full audio chain.
- It is the small sibling of the MDF: **one serial input, one filter path**, the 24-bit output, the CIC (sinc4 / sinc5) with the adjustable decimation, the reshape and the
  high-pass filters, the gain, plus the SAD and the clock absence detector. The user describes the audio format ("16 kHz microphone, 2 MHz clock"), the module calculates
  the dividers and the filter orders as in the Mdf module.
- The **trigger** of the acquisition can be the SAD (the audio is captured only when a sound is detected: the voice-triggered recording) or a signal from another peripheral.
- The module hides the differences between the families: the number of the serial interfaces, the SAD parameters and the low-power domain differ, the interface stays the same.

What the module is **not**: it does not recognise the speech or the keywords (a model on the top of the samples), and it does not process the audio after the filter.

---

## Concept

| Item | Meaning |
|---|---|
| **Serial interface** | The PDM input (`ADF_SITF_PDM`, `SPI`, `MANCHESTER`), the clock output divider for the microphone (1 ... 3.2 MHz typical), the sampling edge (a stereo pair shares the line on the two edges). |
| **Filter** | The output rate, the CIC order and the decimation, the reshape filter, the high-pass filter, the gain, the resolution. |
| **Sound activity detector** (`adf_SadConfig_t`) | The mode (the detection of the sound against the noise floor or an absolute threshold), the frame size and the hysteresis, the noise estimation (the learning of the background level), the hangover time after the sound disappears, the event. |
| **Acquisition** | Continuous, single, or triggered by the SAD / an external trigger. |
| **Stream** | The buffer and the callbacks for the samples. |

Rules:

1. `Adf_Init` configures the clock, the serial interface and the pins and leaves the filter stopped. The filter configuration is changed only while it is stopped.
2. `Adf_Set_AcquisitionStart` starts the acquisition (the SAD may be armed first and the acquisition is then started by its trigger). The samples are streamed by the DMA with
   the half and the complete callbacks, as in the Mdf module.
3. The SAD works with the **noise learning** phase (the module measures the background for a configurable time and sets the threshold above it) or with fixed thresholds; its
   event is delivered to the `SadCallback` and can wake the MCU from the Stop mode on the families whose ADF works in the low-power domain.
4. The output rate is calculated and returned as in the Mdf module; an impossible rate is rejected.
5. The overrun, the clock absence and the (optional) saturation are reported by the error callback.
6. All parameters are checked before a register is written.

---

## Supported families

Every family gets its own story, its own tasks and its own branches `Dev/<Family>` and `Releases/<Family>` cut from this
master.

| Family | Branch | Filter | Status |
|---|---|---|---|
| STM32H5 | `Dev/STM32H5` | `ADF1` on the STM32H5E / H5F lines: one serial input (PDM, SPI, Manchester), 2 common clocks, one filter path (CIC sinc4 / sinc5, reshape, high-pass, gain), the sound activity detector, the clock absence detector, 24-bit output, DMA | planned (first) |
| STM32U5 | `Dev/STM32U5` | `ADF1` (with `MDF1`) in the low-power domain on the lines that have it | planned |
| STM32U3 | `Dev/STM32U3` | device dependent | planned |
| STM32F4, STM32F7, STM32G4, STM32H7, STM32L4 | `Dev/STM32F4` ... | no ADF on the device - the module is a stub that returns `ADF_REQUEST_ERROR` | planned |

A function without a meaning in a family stays in the interface, returns `ADF_REQUEST_ERROR` and suppresses the unused
parameters (for example a SAD parameter the detector of the device does not have).

---

## Public interface in short (proposal)

| Group | Functions |
|---|---|
| Module management | `Adf_Get_ModuleVersion`, `Adf_Init`, `Adf_Deinit`, `Adf_Task`, `Adf_Get_DefaultConfig` |
| Settings | `Adf_Set_SitfConfig`, `Adf_Set_ClockConfig`, `Adf_Set_FilterConfig`, `Adf_Get_OutputRate`, `Adf_Set_Gain` |
| Acquisition | `Adf_Set_AcquisitionStart`, `Adf_Set_AcquisitionStop`, `Adf_Get_Sample`, `Adf_Get_FilterState` |
| Streaming | `Adf_Set_StreamStart`, `Adf_Set_StreamStop`, `Adf_Get_XferError` |
| Sound activity detector | `Adf_Set_SadConfig`, `Adf_Set_SadActive`, `Adf_Set_SadInactive`, `Adf_Get_SadState`, `Adf_Get_NoiseLevel` |
| Interrupts | `Adf_Set_IrqPriority` / `Get`, the callbacks are a part of the configuration |
| Pins | `Adf_InitSdGpio`, `Adf_InitClockGpio` |

```c
adf_ModuleVersion_t Adf_Get_ModuleVersion     ( void );

adf_RequestState_t  Adf_Init                  ( const adf_Config_t * const adfConfig );
adf_RequestState_t  Adf_Deinit                ( void );
void                Adf_Task                  ( void );
adf_RequestState_t  Adf_Get_DefaultConfig     ( adf_Config_t * const adfConfig );

adf_RequestState_t  Adf_Set_FilterConfig      ( const adf_FilterConfig_t * const filterConfig );
adf_RequestState_t  Adf_Set_AcquisitionStart  ( void );
adf_RequestState_t  Adf_Set_AcquisitionStop   ( void );
adf_RequestState_t  Adf_Set_StreamStart       ( const adf_StreamRequest_t * const streamRequest );

adf_RequestState_t  Adf_Set_SadConfig         ( const adf_SadConfig_t * const sadConfig );
adf_RequestState_t  Adf_Set_SadActive         ( void );
adf_RequestState_t  Adf_Get_SadState          ( adf_SadState_t * const sadState );
adf_RequestState_t  Adf_Get_NoiseLevel        ( adf_Level_t * const noiseLevel );
```

Every function returns `ADF_REQUEST_OK` / `ADF_REQUEST_ERROR` and checks its parameters and the state of the module
before it touches a register.

| Type | Meaning |
|---|---|
| `adf_Config_t` | Kernel clock source, the serial interface, the filter, the SAD, transfer mode, DMA channel, pins, callbacks, interrupt priority |
| `adf_SadConfig_t` | Mode, frame size, hysteresis, the noise learning time, the thresholds, the hangover |
| `adf_SadState_t` | `ADF_SAD_IDLE`, `ADF_SAD_LEARNING`, `ADF_SAD_MONITORING`, `ADF_SAD_SOUND_DETECTED` |
| Callbacks | `StreamHalfCallback`, `StreamCompleteCallback`, `SadCallback( state )`, `ErrorCallback` |

---

## Usage

### A voice-activated recorder that sleeps until a sound appears

```c
#include "Adf_Port.h"

static adf_Sample_t recording[ 2u * 320u ];

static void App_SoundDetected( adf_SadState_t sadState ) { /* the MCU is awake: start the full audio chain */ }

adf_Config_t       adfConfig;
adf_FilterConfig_t filterConfig;
adf_SadConfig_t    sadConfig;

(void)Adf_Get_DefaultConfig( &adfConfig );
adfConfig.Sitf.Mode      = ADF_SITF_PDM;
adfConfig.Sitf.ClockHz   = 2048000u;
adfConfig.SadCallback    = App_SoundDetected;
(void)Adf_Init( &adfConfig );

filterConfig.OutputRate  = 16000u;
filterConfig.CicOrder    = ADF_CIC_SINC4;
(void)Adf_Set_FilterConfig( &filterConfig );

sadConfig = adfConfig.Sad;                                   /* the default: noise learning for 1 s, then the detection above it */
(void)Adf_Set_SadConfig( &sadConfig );
(void)Adf_Set_SadActive();
/* enter the Stop mode: the SAD event wakes the MCU, then Adf_Set_StreamStart( &stream ) records the sound */
```

---

## Behaviour and limits

- **Noise learning.** The SAD needs a quiet environment during the learning; a sound in this phase raises the threshold (the module can repeat the learning on request).
- **Low power.** The SAD works in the low-power modes only on the families whose ADF is in the low-power domain and only with the kernel clock that runs there; the consumption
  is dominated by the microphone and its clock.
- **Settling.** The filters have the settling time after the start; the first samples are discarded.
- **Not part of the first version:** the keyword spotting, the automatic gain control, the two-microphone beam-forming, the sound classification.

---

## Dependencies

- **Rcc** - peripheral clock, the kernel clock source and its frequency, the clock request in the low-power modes
- **Nvic** and **Exti** - the interrupt and the wake-up line of the SAD
- **Gpio** - the serial data and clock pins
- **Gpdma** - DMA transfers of the samples
- **RAL (Register Abstraction Layer)** - used internally to access the registers

All mandatory RCC, NVIC, EXTI, GPIO and DMA configurations are handled internally.

---

## Testing

- **Unit tests** (host, Unity + CMock + RegMem): the output rate search, the filter and the SAD configuration to register mapping, the SAD state machine (learning, monitoring,
  sound, hangover), the stream flow, the rejected changes of a running filter.
- **Integration tests** (target, probe-rs): a generated PDM stream (a timer / the GPIO modulator) converted and compared, the SAD detection of a tone burst against the noise
  floor, the wake-up from the Stop mode.

---

## Interface conventions

The module follows the common rules of the MCAL layer, so that it behaves like the other modules:

- **Lists instead of free members.** Everything that the hardware restricts (the pins, the kernel clock sources, the DMA
  channels, the triggers and the other selectable sources) is a `typedef enum` list with exactly the items that are valid
  for the device line; a pin is an item `ADF_<SIGNAL>_PIN_<INSTANCE>_P<port><pin>` and there is an `*_UNUSED` item. A choice
  that is not in the list cannot be written.
- **Own peripheral only.** The module calls only the registers / LL functions of its own peripheral. The clocks, the
  interrupts, the pins, the DMA and the EXTI lines go through the API of the Rcc, Nvic, Gpio, Gpdma and Exti modules; a
  missing function is added to that module, it is not bypassed.
- **Init verifies itself.** `Adf_Init` checks the result at the end; when it is not as required, the module is
  deinitialized and an error is returned. The waiting for the hardware is a loop on the register state with a timeout,
  never a blind delay.
- **Transfer modes.** Where the module moves data, the POLL (served by `Adf_Task`), ISR and DMA modes have the same
  configuration structure, the same request and the same callbacks (error, half transfer, transfer complete).
- **Flags and states are enums of the module** (`adf_FunctionState_t`, `adf_FlagState_t`), the limits and the constants are
  named, there are no magic numbers in the interface.
- **Unsupported functions stay.** A function without a meaning in a family keeps its signature, suppresses the unused
  parameters and returns `ADF_REQUEST_ERROR`.
- **Documentation.** Every function and type has a Doxygen description, the main header has the `\defgroup`.

---

## Repository layout and branches

The master holds this README and the license. The family branches (`Dev/<Family>`, `Releases/<Family>`) hold:

| Path | Content |
|---|---|
| `README.md` | README of the family: the instances of the device, specifics, errata, status |
| `Adf_Port.h`, `Adf_Types.h` | Public interface (identical in all families) |
| `Adf*.c`, `Adf.h` | Implementation of the family |
| `CMakeLists.txt` | Library `Adf_Lib` of the EmBi platform |
| `Tests/UnitTests` | Unit tests (Unity + CMock + RegMem) |
| `Tests/IntegrationTests` | Integration tests on the target (probe-rs) |

A change of the public interface is made in all families.

---

## CMake integration

```cmake
target_link_libraries( User_Lib PRIVATE Adf_Lib )
```

1. Link `Adf_Lib` to your CMake library.
2. Include `Adf_Port.h` in your project.
3. Configure the module as needed for your hardware.

---

## License

This project is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)**.

You are free to use, modify, and share this work for **non-commercial purposes**, provided appropriate credit is given.

See [LICENSE.md](LICENSE.md) for full terms or visit [creativecommons.org/licenses/by-nc/4.0](https://creativecommons.org/licenses/by-nc/4.0/).

---

## Authors

- **Mr.Nobody** - [embedbits.com](https://embedbits.com)

Contributions are welcome! Please open a pull request.
