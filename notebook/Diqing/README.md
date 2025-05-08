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
    - Observed ~50 ms jitter in Notify timing (to investigate later).

---

# 2025-03-24 - ML Model Research (Research)

- Surveyed EMG classification papers:
    - Focused on architectures suited for low-resource MCUs,
    - Key reference: Vasconez et al. (2022) - EMG-IMU + CNN,
    - Compared feature-engineered methods vs. end-to-end CNNs.
- Noted STM32WB resource limits (1 MB Flash, 256 KB RAM) likely constrain CNN usage.

# 2025-03-27 - CNN Prototype (Implementation & Testing)

- Implemented CNN on PC (Python + Keras):
    - 2 Conv layers (ReLU) + Dense + Softmax,
    - Input: 4-channel EMG, 2 sec @ 2 kHz = (4 × 4000) samples.
- Accuracy: ~85% (single-subject, 30 trials),
- Model size: ~300 KB quantized → too large for STM32WB.
- Logged findings and pivot plan:
    - Explore feature-based model next,
    - Focus on minimizing Flash + RAM footprint.

---

# 2025-03-31 - Feature Model Dev (Implementation)

- Designed new ML pipeline:
    - Per channel: Mean, Std Dev, RMS, MAV, Zero Crossings,
    - 4 channels × 5 features = 20D input vector.
- PC-side code:
    - `compute_features(emg_data)` to extract features,
    - Keras classifier: 2 Dense layers (50 neurons each) + Softmax output.
- Results:
    - ~90% accuracy (30 trials, single user),
    - Model size ~180 KB (quantized), acceptable for MCU.
- Documented next step: embed feature extraction code into STM32 firmware.

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
    - First run: saw incorrect data alignment (fixed by adjusting SPI word size),
    - Added `printf` logs + toggled debug LED for data-ready check.

# 2025-04-07 - BLE Debugging (Debugging)

- BLE began hanging at `APP_BLE_Init()`,
    - RTC, RF, IPCC initialized,
    - System stalled at `aci_gap_init()`.
- Added debug LED blink after each init step to trace progress.
- Hypothesis: BLE stack binary corrupted.
- Shifted focus: prioritized USB-CDC logging as **main method** moving forward.

---

# 2025-04-10 - P2P Validation (Debugging & Verification)

- Flashed official STM32 P2P Server example (unmodified),
    - Result: BLE still hung at `aci_gap_init()`.
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