# Diqing Worklog

[[_TOC_]]

# 2025-02-12 - Discussion with TA (Research & Planning)

- Met with TA Michael to review the initial Project Proposal draft.
- Action items:
    - Expand Subsystem Overviews: BLE (STM32WB’s BLE stack) and Processing (EMG acquisition + ML pipeline),
    - Add verification steps for BLE throughput, ADC sampling, and power budget.
- Skimmed STM32WB5MMG-DK user manual (UM2825) to check BLE block diagram and capabilities.

# 2025-02-13 - Project Proposal Updates (Research & Drafting)

- Coordinated Overleaf edits:
    - BLE: planned to use STM32_WPAN middleware with BLE 5.0 (2 Mbps PHY),
    - Processing: 2 kHz EMG acquisition (4 channels) via STM32 ADC + SPI.
- Documented that firmware flashing would need USB DFU mode:
    - **Short CN13 GND + BOOT0 pin while powering on**,
    - Noted no ST-Link flashing allowed when BLE is enabled.
- Drafted subsystem verification plan:
    - BLE: test via nRF Connect,
    - EMG: log data to PC and check waveform.

---

# 2025-02-18 - Final Proposal Submission (Research Wrap-Up)

- Did a final consistency check between hardware (STM32WB5MMG-DK) and proposal requirements.
- BLE: confirmed P2P Server template is a recommended starting point (ST Wiki).
- Processing: planned TensorFlow Lite Micro deployment after model training.

# 2025-02-22 - BLE Stack Prep (Planning & Early Setup)

- Deep-dived into STM32 BLE stack:
    - CPU1 (App) ↔ CPU2 (BLE radio),
    - Key peripherals: RTC (timers), RF, IPCC (comm link).
- Documented USB-CDC interface plan for logging and fallback.
- Sketched initial BLE service layout:
    - Custom service + Notify characteristic,
    - Payload: 1-byte header, 2-byte data, 1-byte checksum.

---

# 2025-03-10 - BLE Stack Setup (Implementation)

- Set up CubeMX project:
    - Enabled BLE middleware, RTC, RF, IPCC, USB-CDC,
    - Checked clock tree: HSE (32 MHz), LSE (32.768 kHz) configured,
    - Allocated BLE stack buffers per STM32CubeMX defaults.
- Flashed firmware via USB DFU:
    - Shorted CN13 pins (GND + BOOT0) → powered on → STM32CubeProgrammer.
- User code:
    - Implemented `APP_BLE_Init()`: BLE advertising started,
    - Set device name: `"Team57-EMG"` using `aci_gap_set_discoverable()`.

# 2025-03-12 - BLE Notify Service (Implementation & Verification)

- Built custom BLE service:
    - Added Notify characteristic with correct UUID and attribute permissions,
    - Used P2P Server template as base.
- User code:
    - Wrote `BLE_App_SendNotification()`: assembled payload, called `aci_gatt_update_char_value()`,
    - Configured periodic RTC wakeup (200 ms) to trigger Notify.
- Test:
    - Connected via nRF Connect (iPhone + PC),
    - Verified periodic Notify packets; data matched expected mock pattern.
- Issues:
    - Observed ~50 ms jitter in Notify timing (noted for later investigation).

---

# 2025-03-24 - ML Model Research (Research)

- Surveyed EMG classification papers:
    - Focused on low-resource architectures for embedded systems,
    - Key reference: Vasconez et al. (2022) - CNN-based EMG classification,
    - Compared CNNs, RNNs, and feature-based MLPs.
- Identified STM32WB5MMG resource constraints:
    - 1 MB Flash, 256 KB RAM → tight for large models,
    - Need ≤ 200 KB quantized model size for practical deployment.
- Planned to prototype a small CNN first to benchmark feasibility.

# 2025-03-27 - CNN Prototype (Implementation & Testing)

- **Model 1: Small CNN**
    - Architecture:
        - Input: (4 channels × 4000 samples),
        - Conv1D (32 filters, kernel=5) → ReLU,
        - Conv1D (64 filters, kernel=5) → ReLU,
        - Flatten → Dense (50) → Softmax (6 classes).
    - Accuracy: ~85% (30 trials, single user),
    - Model size:
        - ~1.2 MB (float32),
        - ~300 KB (after full quantization).
- Issues:
    - STM32WB5MMG could not fit 300 KB fully quantized model,
    - Overfitting observed despite dropout (0.2).
- Decision:
    - Too large; planned to test a smaller CNN next.

---

# 2025-03-29 - CNN Lite Experiment (Implementation & Testing)

- **Model 2: Lite CNN**
    - Architecture:
        - Input: (4 channels × 4000 samples),
        - Conv1D (16 filters, kernel=3) → ReLU,
        - GlobalMaxPooling1D,
        - Dense (30) → ReLU → Dropout(0.2),
        - Softmax (6 classes).
    - Accuracy: ~81%,
    - Model size:
        - ~550 KB (float32),
        - ~150 KB (quantized).
- Notes:
    - Fit within MCU constraints,
    - But accuracy trade-off (~9% lower).

---

# 2025-03-31 - Feature-Based MLP (Implementation)

- **Model 3: Feature-Based MLP**
    - Feature extraction (per channel):
        - Mean, Std Dev, RMS, MAV, Zero Crossings,
    - Input: 20D vector (5 features × 4 channels),
    - Architecture:
        - Dense (50) → ReLU + BatchNorm + Dropout(0.2),
        - Dense (50) → ReLU + BatchNorm + Dropout(0.2),
        - Softmax (6 classes).
    - Accuracy: ~90% (30 trials),
    - Model size:
        - ~480 KB (float32),
        - ~110 KB (quantized).
- Advantages:
    - Small, fast, deployable,
    - Outperformed CNNs in accuracy and size balance.

# 2025-04-02 - Hardware Integration (Implementation & Testing)

- Received breadboard with **ADS1198 ADC** circuit from teammates.
- Wired STM32WB5MMG-DK:
    - SPI1: SCLK, MISO, MOSI, CS to ADS1198,
    - Verified pin mapping against STM32WB datasheet + breadboard schematic.
- CubeMX:
    - Configured SPI1 (Master, 2 MHz clock),
    - Enabled GPIO for ADC_DRDY interrupt (planned for later).
- Verified power rail stability with multimeter.

---

# 2025-04-04 - ADC SPI Testing (Implementation & Debugging)

- Wrote `read_ads1198()`:
    - Pulled CS low via `HAL_GPIO_WritePin()`,
    - Called `HAL_SPI_TransmitReceive()` to fetch 4-channel EMG data.
- Added `usb_stream()` to send data to PC via USB-CDC (`CDC_Transmit_FS()`).
- Verified:
    - Live EMG data on PC (RealTerm + Python),
    - Sampling rate: stable at 2 kHz.
- Debugging:
    - Initial bug: data alignment issue (fixed SPI word size),
    - Added `printf` logs via **USART2 (UART peripheral)** to output raw values for verification,
    - Config: USART2 at 115200 bps, 8N1 (PA2/PA3),
    - Used logic analyzer to verify SPI signals.

# 2025-04-07 - BLE Debugging (Debugging)

- BLE stack began hanging at `APP_BLE_Init()`.
- **USART2 Debug Setup:**
    - Added `__io_putchar()` retarget for printf:
    ```c
    int __io_putchar(int ch)
    {
        HAL_UART_Transmit(&huart2, (uint8_t*)&ch, 1, HAL_MAX_DELAY);
        return ch;
    }
    ```
    - Printed checkpoints:
        - `"RTC Init Done"`,
        - `"RF Init Done"`,
        - `"IPCC Init Done"`,
        - `"aci_gap_init() Start"`.
    - Used RealTerm to confirm serial output.
- **Breakpoints & Stepping:**
    - Set breakpoints at:
        - `APP_BLE_Init()`,
        - `SHCI_C2_BLE_Init()`,
        - `aci_gap_init()`,
    - Stepped through using STM32CubeIDE: stuck after `aci_gap_init()`.
- Debug LEDs:
    - Toggled PB0 LED after each init step,
    - LED froze after BLE stack call → confirmed stall point.
- Conclusion:
    - BLE stack binary corruption likely; pivoted to USB-CDC as primary path.

---

# 2025-04-10 - P2P Validation (Debugging & Verification)

- Flashed official STM32 P2P Server example,
    - USART2 logs + breakpoints showed identical failure point,
    - Confirmed BLE stack binary issue (vendor side).
- Confirmed BLE stack binary was faulty (not an app-side bug).
- Documented issue + planned to continue with USB-CDC only until vendor fix.

# 2025-04-14 - End-to-End Pipeline Test (Implementation & Verification)

- Full firmware pipeline:
    - `read_ads1198()` (SPI) → `extract_features_live()` → mock inference,
    - Streamed output via `CDC_Transmit_FS()` over USB-CDC.
- Collected 30 new trials,
    - PC-side classifier verified ~90% accuracy.
- Verified:
    - Clean data logs,
    - No missed samples at 2 kHz.

---

# 2025-04-17 - Mock Demo Prep (Testing & Debugging)

- Assembled full system:
    - ADS1198 (SPI1), STM32WB5MMG-DK, USB-CDC streaming (primary method),
    - BLE Notify code still in firmware (via `aci_gatt_update_char_value()`) but inactive due to broken stack.
- Verified data stability:
    - 60 min run: no crashes,
    - ~5 ms latency on USB-CDC link (PC timestamps).
- Prepared demo script explaining BLE issue and showing live EMG + classification.

# 2025-04-21 - Mock Demo (Verification)

- Demo run:
    - Live EMG acquisition (2 kHz),
    - USB-CDC data stream to PC,
    - PC: `compute_features() + model.predict()` in real time,
    - BLE non-functional; logged and explained root cause.

---

# 2025-04-25 - BLE Stack Update (Debugging)

- Updated STM32WB FUS + BLE stack to latest release,
- Re-flashed P2P Server for validation: still failed (hang at `SHCI_C2_BLE_Init()`),
- Concluded BLE broken in vendor stack; documented and closed out BLE testing.

# 2025-04-28 - USB-CDC Optimization (Implementation & Verification)

- Optimized USB code:
    - Increased `APP_RX_DATA_SIZE` in `usbd_cdc_if.c`,
    - Tuned buffer flushing in `CDC_Transmit_FS()`.
- PC-side:
    - Added timestamp logging (Python) to monitor delays,
    - Verified <5 ms typical delay.
- Ran extended tests:
    - 2-hour streaming: stable, no packet loss.

---

# 2025-05-02 - Final Demo (Verification)

- Delivered final demo:
    - EMG acquisition (2 kHz via `read_ads1198()`),
    - USB-CDC streaming (`CDC_Transmit_FS()`),
    - PC classifier (~90% accuracy),
    - Highlighted BLE issue and fallback to USB as primary method.

# 2025-05-07 - Final Report Submission (Documentation)

- Finalized:
    - Processing: full pipeline, model design, STM32 user code (e.g., `read_ads1198()`, `CDC_Transmit_FS()`),
    - BLE: initial success, final failure; documented flashing method (CN13 shorting, USB-only),
    - Ethics & Safety, Testing logs.
- Submitted final report.