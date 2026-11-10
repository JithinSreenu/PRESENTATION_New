# Prosthetic Knee Telemetry System — Simplified Report

> **Author:** Jithin Sreenu  
> **Date:** 09 November 2026  
> **Firmware:** STM32U585 (C / FreeRTOS) &nbsp;|&nbsp; **Desktop App:** Prosthetic Telemetry Studio v1.3.0 (Electron)

---

## 1. What Is This Project?

This project is a **complete real-time monitoring system for a prosthetic knee joint**. It has two main parts:

| Part | What It Does | Built With |
|------|-------------|------------|
| **Firmware** (runs on the microcontroller) | Reads sensors on the prosthetic knee (force, angle, battery), detects the walking phase, and sends data 100 times per second | C language, FreeRTOS, STM32CubeIDE |
| **Desktop App** (runs on a PC/laptop) | Receives the data, shows live charts & graphs, records sessions, and lets clinicians tune the knee settings | Electron (JavaScript), Plotly charts |

> [!NOTE]
> Think of it like a **heart rate monitor** but for a prosthetic knee — it continuously measures forces, angles, and walking phases so clinicians can tune the prosthesis for the patient.

---

## 2. System Architecture — The Big Picture

This diagram shows how all the pieces connect:

```mermaid
flowchart LR
    subgraph PROSTHETIC_KNEE["🦿 Prosthetic Knee"]
        LC1["Load Cell 1\n(Force Sensor)"]
        LC2["Load Cell 2\n(Force Sensor)"]
        KA["Knee Angle\nSensor"]
        BAT["Battery\nVoltage"]
        VALVE["Spool Valve\n(Motor Slave)"]
    end

    subgraph MCU["🔲 STM32U585 Microcontroller"]
        ADC["ADC\n(Analog-to-Digital)"]
        FIR["FIR Filter\n(Noise Removal)"]
        SM["Gait State Machine\n(Walking Phase)"]
        PROTO["Protocol Builder\n(25-byte frame)"]
        ENC["AES-256 Encryption\n(Optional)"]
        SLAVE["Slave Link\n(USART3)"]
    end

    subgraph PC["💻 Desktop Computer"]
        SERIAL["Serial Port\n(USB / BLE)"]
        PARSER["Frame Parser\n& CRC Check"]
        UI["Live Dashboard\n& Charts"]
        REC["Session Recorder\n(CSV / JSON)"]
        CONFIG["Configuration\nUploader"]
    end

    LC1 --> ADC
    LC2 --> ADC
    KA --> ADC
    BAT --> ADC
    ADC --> FIR --> SM --> PROTO --> ENC
    ENC -->|"USART1\n115200 baud\n100 Hz"| SERIAL
    SERIAL --> PARSER --> UI
    PARSER --> REC
    CONFIG -->|"CLI commands\n& config packets"| ENC
    SLAVE <-->|"7-byte frames\nSWING/STANCE"| VALVE
```

---

## 3. Hardware Used

```mermaid
flowchart TD
    subgraph BOARD["ST B-U585I-IOT02A Discovery Board"]
        CPU["STM32U585AI\nARM Cortex-M33\n160 MHz, 768 KB RAM"]
        ADC1["ADC1 (12-bit)\n6 channels"]
        DMA["GPDMA1\n(Direct Memory Access)"]
        U1["USART1 (to PC)\n115200-8N1"]
        U3["USART3 (to Slave)\nPD8-TX / PD9-RX"]
        AES_HW["Hardware AES\nPeripheral"]
        TRNG["True Random\nNumber Generator"]
        FLASH["Internal Flash\nBank 2 (settings)"]
    end

    subgraph PINS["Sensor Pin Connections"]
        A0["PC0 (A0) → Load Cell 1"]
        A1["PC2 (A1) → Load Cell 2"]
        A2["PC4 (A2) → Battery"]
        A3["PC5 (A3) → Reserved"]
        A4["PA7 (A4) → Reserved"]
        A5["PB0 (A5) → Reserved"]
    end

    ADC1 --> DMA
    CPU --> U1
    CPU --> U3
    CPU --> AES_HW
    CPU --> TRNG
    CPU --> FLASH
```

> [!IMPORTANT]
> The board uses **3 active sensor channels** (Force, Knee Angle, Battery). The other 3 Arduino-header channels (A3–A5) are configured but reserved for future use.

---

## 4. Firmware Data Pipeline — Step by Step

This is what happens inside the microcontroller every 10 milliseconds:

```mermaid
flowchart TD
    A["⏱ 100 Hz Timer Fires\n(every 10 ms)"] --> B["📡 ADC Scans 3 Channels\nvia DMA (32 samples each)"]
    B --> C["🔢 96 raw values arrive\nin memory automatically"]
    C --> D["🧹 51-tap FIR Filter\nremoves noise (25 Hz cutoff)"]
    D --> E["📐 Scale to\nEngineering Units"]
    E --> F["🦶 Gait State Machine\ndetects walking phase"]
    F --> G["📦 Build 25-byte\nTelemetry Frame"]
    G --> H{"🔒 Encryption\nEnabled?"}
    H -->|Yes| I["🔐 AES-256-GCM\nEncrypt → 31 bytes"]
    H -->|No| J["📤 Send 25 bytes\nvia USART1 DMA"]
    I --> J

    style A fill:#4CAF50,color:#fff
    style H fill:#FF9800,color:#fff
    style J fill:#2196F3,color:#fff
```

### Scaling Factors (ADC → Real-World Values)

| Sensor | Raw ADC | Scale Factor | Output Unit |
|--------|---------|-------------|-------------|
| Load Cell 1 (Force) | 0–4095 (12-bit) | × 300 | Newtons (N) |
| Load Cell 2 (Moment) | 0–4095 | × 60 | Newton-metres (Nm) |
| Knee Angle | 0–4095 | × 27 | Degrees (°) |
| Battery | 0–4095 | × 1.41 | Volts (V) |

---

## 5. Telemetry Frame Format

Every 10 ms, the firmware sends this exact 25-byte packet to the PC:

```
Byte:  [0] [1] [2] [3-4]   [5-6]    [7-8]   [9-10]  [11-12] [13]  [14-15] [16-22] [23-24]
       ─── ─── ─── ─────── ──────── ─────── ─────── ─────── ───── ─────── ─────── ───────
       AA  55  seq  Force   Moment   Knee    Valve   Battery State Fault   Padding  CRC-16
       ▲   ▲   ▲    i16×100 i16×100  i16×100 u16     u16×100 u8    u16     zeros   MODBUS
       │   │   │
       └───┴───┘
       Sync bytes (always 0xAA 0x55)
```

> [!TIP]
> All multi-byte values use **little-endian** byte order and **×100 fixed-point** encoding. For example, a force of 123.45 N is sent as the integer 12345.

---

## 6. Gait State Machine — Walking Phase Detection

The state machine identifies which phase of the walking cycle the patient is in. This is the core clinical intelligence of the system.

### 6.1 The 6 Gait Phases

```mermaid
stateDiagram-v2
    [*] --> SWING: Boot / Reset

    SWING --> LOADING: Force rises\ninto loading band
    LOADING --> MID_STANCE: Force enters\nmid-stance band
    MID_STANCE --> TERMINAL_STANCE: Force falls back\nto loading band\n(mid-stance was seen)
    TERMINAL_STANCE --> PRE_SWING: Force drops to\npre-swing band
    PRE_SWING --> SWING: Force drops to\nswing band

    SWING --> STANCE: Force above\nstance threshold\n(no specific band match)

    note right of SWING: Leg is in the air\n(no ground contact)
    note right of LOADING: Foot just hit the ground\n(weight loading)
    note right of MID_STANCE: Full weight on leg\n(standing over foot)
    note right of TERMINAL_STANCE: Weight shifting forward\n(pushing off)
    note right of PRE_SWING: About to lift off\n(toe push)
    note left of STANCE: Generic loaded state\n(catch-all)
```

### 6.2 How It Decides (Decision Logic)

```mermaid
flowchart TD
    START["New sensor window\ncomplete (N cycles)"] --> COMBINE["Combine 2 load cells\n(SUM / AVG / MAX / MIN)"]
    COMBINE --> MOMENT["Calculate Moment\n= Force × Arm (0.2m)"]
    MOMENT --> SELECT{"Decision Input\nMode?"}

    SELECT -->|FORCE| USE_F["Use combined\nforce value"]
    SELECT -->|MOMENT| USE_M["Use calculated\nmoment value"]
    SELECT -->|FORCE+KNEE| USE_FK["Use force value\n+ knee angle"]

    USE_F --> BANDS
    USE_M --> BANDS
    USE_FK --> BANDS_FK["Check force bands\nAND knee angle bands\nsimultaneously"]

    BANDS["Check value against\nphase bands"] --> SW{"In Swing\nband?"}
    SW -->|Yes| SET_SWING["State = SWING\nClear mid_seen flag"]
    SW -->|No| PS{"In Pre-Swing\nband?"}
    PS -->|Yes & mid_seen| SET_PS["State = PRE-SWING"]
    PS -->|No| LO{"In Loading\nband?"}
    LO -->|"Yes & !mid_seen"| SET_LO["State = LOADING"]
    LO -->|"Yes & mid_seen"| SET_TS["State = TERMINAL-STANCE"]
    LO -->|No| MS{"In Mid-Stance\nband?"}
    MS -->|Yes| SET_MS["State = MID-STANCE\nSet mid_seen = 1"]
    MS -->|No| ST{"Force ≥\nstance min?"}
    ST -->|Yes| SET_ST["State = STANCE"]
    ST -->|No| KEEP["Keep previous state\n(hysteresis)"]

    BANDS_FK --> SW

    style SET_SWING fill:#4CAF50,color:#fff
    style SET_LO fill:#FF9800,color:#fff
    style SET_MS fill:#F44336,color:#fff
    style SET_TS fill:#9C27B0,color:#fff
    style SET_PS fill:#3F51B5,color:#fff
    style SET_ST fill:#795548,color:#fff
    style KEEP fill:#607D8B,color:#fff
```

> [!NOTE]
> **The `mid_seen` flag** is the key sequencing trick: LOADING and TERMINAL-STANCE share the same force band, but LOADING can only happen *before* mid-stance is seen, and TERMINAL-STANCE can only happen *after*. The flag resets on every SWING.

### 6.3 Walking Cycle Diagram (Simplified)

```mermaid
flowchart LR
    subgraph ONE_STRIDE["One Complete Walking Stride"]
        direction LR
        S1["🦶 SWING\n(leg in air)"]
        S2["👟 LOADING\n(heel strike)"]
        S3["🧍 MID-STANCE\n(full weight)"]
        S4["🏃 TERMINAL\n(push off)"]
        S5["💨 PRE-SWING\n(toe off)"]
        S6["🦶 SWING\n(next cycle)"]
    end

    S1 -->|"Force ↑"| S2
    S2 -->|"Force ↑↑"| S3
    S3 -->|"Force ↓"| S4
    S4 -->|"Force ↓↓"| S5
    S5 -->|"Force → 0"| S6
```

---

## 7. Slave Motor Link

The microcontroller also controls a **spool valve motor** on the prosthetic knee via a second UART:

```mermaid
sequenceDiagram
    participant MCU as STM32 Master<br/>(USART3 TX/RX)
    participant SLAVE as Motor Slave<br/>(Spool Valve)

    Note over MCU,SLAVE: On every gait state transition...

    MCU->>SLAVE: 7-byte command frame<br/>(SWING=0 or STANCE=1)
    SLAVE-->>MCU: 7-byte response<br/>(current valve position %)

    Note over MCU: Median-of-3 filter on valve data<br/>25 ms freshness gate<br/>CRC-validated

    MCU->>SLAVE: Next state command
    SLAVE-->>MCU: Updated valve position
```

> The slave converts SWING/STANCE commands into hydraulic valve positions that control the knee's resistance during walking.

---

## 8. Fault Detection System

The firmware continuously monitors for problems and reports them as a 16-bit fault register in every frame:

```mermaid
flowchart TD
    subgraph FAULTS["16-bit Fault Bitmask (bits 0-15)"]
        direction TB
        B0["Bits 0-9: Threshold\nLimit Violations\n(min/max per channel)"]
        B1["Bits 10-13: Per-Channel\nSensor Failures\n(stuck-zero, stuck-VREF,\nflatline)"]
        B2["Bit 14: ADC Stuck\nat Zero"]
        B3["Bit 15: ADC Stuck\nat VREF"]
    end

    subgraph CHECKS["Continuous Checks"]
        C1["Is force > max?\nor < min?"]
        C2["Is sensor reading\n~0 for > 0.5s?"]
        C3["Is sensor reading\n~3.3V for > 0.5s?"]
        C4["Is variance ~0\nfor > 1.5s?"]
    end

    C1 --> B0
    C2 --> B1
    C3 --> B1
    C4 --> B1
```

---

## 9. AES-256-GCM Encryption (Optional)

When encryption is enabled, every telemetry frame gets encrypted before transmission:

```mermaid
flowchart TD
    subgraph SETUP["🔑 One-Time Key Setup"]
        PSK["Pre-Shared Key (PSK)\n32 bytes / 64 hex chars\nstored in Flash"]
        NONCE["16-byte Nonce\n(from HW RNG or fallback)"]
        SK["Session Key = \nAES-ECB(PSK, Nonce) ||\nAES-ECB(PSK, Nonce⊕0x01)"]
        PSK --> SK
        NONCE --> SK
    end

    subgraph PER_FRAME["🔐 Per-Frame Encryption"]
        PT["Plaintext: bytes 3-22\n(20 bytes of sensor data)"]
        IV["IV = Nonce[0..8] ||\ncounter (big-endian 32-bit)"]
        AAD["AAD = AA 55 seq\n(3 header bytes)"]
        GCM["AES-256-GCM\nEncrypt"]
        CT["Ciphertext (20 bytes)\n+ Tag (8 bytes, MSB)"]
        V2["v2 Frame: AA 55 seq |\nCT[20] | TAG[8]\n= 31 bytes total"]

        PT --> GCM
        IV --> GCM
        AAD --> GCM
        SK --> GCM
        GCM --> CT --> V2
    end

    style PSK fill:#F44336,color:#fff
    style V2 fill:#4CAF50,color:#fff
```

| Plain Frame (v1) | Encrypted Frame (v2) |
|---|---|
| 25 bytes | 31 bytes |
| AA 55 seq \| data \| CRC | AA 55 seq \| ciphertext[20] \| tag[8] |

---

## 10. Desktop App (Prosthetic Telemetry Studio)

### 10.1 App Architecture

```mermaid
flowchart TD
    subgraph ELECTRON["Electron App (3 processes)"]
        MAIN["Main Process\n(Node.js)\n• Window management\n• Serial port (node-serialport)\n• File save dialogs\n• IPC bridge"]
        PRELOAD["Preload Script\n(contextBridge)\n• Exposes safe API\n• No Node in renderer"]
        RENDERER["Renderer Process\n(Chromium)\n• HTML/CSS/JS UI\n• Plotly charts\n• WebGL 3D view\n• Protocol parser"]
    end

    subgraph TRANSPORTS["Connection Methods"]
        USB["USB Serial\n(node-serialport)"]
        WEB["Web Serial API\n(fallback)"]
        BLE["Bluetooth LE\n(Web Bluetooth, NUS)"]
        DEMO["Demo Mode\n(100 Hz synthetic gait)"]
    end

    USB --> MAIN
    WEB --> RENDERER
    BLE --> RENDERER
    DEMO --> RENDERER
    MAIN <-->|"IPC\n(contextBridge)"| PRELOAD <--> RENDERER

    style MAIN fill:#2196F3,color:#fff
    style RENDERER fill:#4CAF50,color:#fff
    style PRELOAD fill:#FF9800,color:#fff
```

### 10.2 App Features — Tab Overview

The app has **16 tabs**. Here's what each one does:

```mermaid
flowchart LR
    subgraph MONITORING["📊 Monitoring"]
        T1["Overview\nDashboard"]
        T2["Live\nWaveforms"]
        T3["Oscilloscope"]
        T4["3D State\nSpace"]
    end

    subgraph ANALYSIS["📈 Analysis"]
        T5["Gait\nAnalytics"]
        T6["Frame\nInspector"]
        T7["XY Plot"]
    end

    subgraph CONTROL["⚙️ Control"]
        T8["Connection\n(USB/BLE)"]
        T9["Thresholds"]
        T10["Coefficients\n(Calibration)"]
        T11["Firmware\nConfig"]
    end

    subgraph RECORD["💾 Recording"]
        T12["Session\nRecorder"]
        T13["Serial\nMonitor"]
    end

    subgraph SECURITY["🔐 Security"]
        T14["Crypto\nSetup"]
        T15["About"]
    end
```

| Tab | What You See | Why It's Useful |
|-----|-------------|-----------------|
| **Overview** | 11 live KPI cards (force, angle, battery, faults, packet rate) | Quick health check at a glance |
| **Waveforms** | 4 scrolling charts (Force, Moment, Knee Angle, Battery) | See trends over time (5–60 s window) |
| **Oscilloscope** | Canvas-rendered glow traces, per-channel | Clean signal view like a lab scope |
| **3D State Space** | WebGL 3D plot (knee × force × moment) coloured by gait phase | Visualize the gait cycle as a 3D trajectory |
| **Gait Analytics** | Cadence, stride time, stance/swing %, peak values, symmetry | Quantitative gait assessment for clinicians |
| **Frame Inspector** | Hex dump of every 25-byte frame with field highlighting | Debug and verify protocol correctness |
| **Connection** | USB port selector, BLE scan, Demo mode toggle | Connect/disconnect from the device |
| **Thresholds** | Min/max sliders for each sensor channel | Set safety limits (triggers fault flags) |
| **Coefficients** | Scale, offset, VREF fields per sensor | Calibrate sensors without reflashing |
| **Firmware Config** | Gait state machine parameters (window, bands, mode) | Tune walking detection live via CLI |
| **Recording** | Start/stop, subject metadata, CSV/JSON export | Capture sessions for later analysis |
| **Serial Monitor** | Raw terminal with quick-command buttons | Direct CLI access to firmware |
| **Crypto** | PSK entry, one-click auto-setup, session status | Enable/disable encrypted link |

---

## 11. End-to-End Data Flow

```mermaid
sequenceDiagram
    participant S as Sensors<br/>(Load Cells, Angle)
    participant ADC as ADC + DMA<br/>(32 samples × 3 ch)
    participant FIR as FIR Filter<br/>(51-tap, 25 Hz)
    participant SM as Gait State<br/>Machine
    participant TX as USART1 TX<br/>(DMA)
    participant PC as Desktop App<br/>(Parser)
    participant UI as Live UI<br/>(Charts)

    Note over S,UI: Every 10 ms (100 Hz cycle)

    S->>ADC: Analog voltages
    ADC->>FIR: 96 raw values (DMA complete)
    FIR->>SM: Filtered, scaled values
    SM->>SM: Determine gait phase
    SM->>TX: Build 25-byte frame
    TX->>PC: Serial bytes (2500 B/s)
    PC->>PC: Find sync (AA 55), parse, CRC check
    PC->>UI: Update charts @ 20 Hz flush rate
```

---

## 12. Firmware FreeRTOS Task Structure

```mermaid
flowchart TD
    subgraph RTOS["FreeRTOS Tasks & Timers"]
        BOOT["Boot Sequence\n• Clock init (160 MHz)\n• Peripheral init\n• Flash load (thresholds, PSK)\n• ADC + DMA start\n• FreeRTOS scheduler start"]
        
        TIMER["100 Hz Software Timer\n(10 ms period)"]
        
        CALLBACK["Timer Callback\n• ADC_Poll()\n• SM_Run()\n• Fault checks\n• Build frame\n• Encrypt (if armed)\n• UART TX via DMA"]

        CLI["CLI Task\n(USART1 RX idle-line)\n• Parse commands\n• mode/demo/set/get\n• crypto/halt/sensors"]
        
        SLAVE_T["Slave RX Service\n(USART3 idle-line DMA)\n• Parse 7-byte frames\n• Median-of-3 filter\n• 25 ms freshness"]
    end

    BOOT --> TIMER
    TIMER -->|"every 10 ms"| CALLBACK
    BOOT --> CLI
    BOOT --> SLAVE_T
```

---

## 13. Configuration & Calibration Flow

```mermaid
sequenceDiagram
    participant Clinician as 👨‍⚕️ Clinician
    participant App as Desktop App
    participant FW as STM32 Firmware
    participant Flash as Internal Flash

    Note over Clinician,Flash: Threshold Configuration

    Clinician->>App: Set min/max for each channel
    App->>FW: 45-byte packet (A5 31 + 10 floats)
    FW->>FW: Apply thresholds (OR fault bits each frame)

    Note over Clinician,Flash: Calibration Coefficients

    Clinician->>App: Enter password
    App->>FW: "halt on" command
    FW->>FW: Stop telemetry (safe state)
    App->>FW: 53-byte packet (A5 32 + 12 floats)
    FW->>Flash: Save coefficients (with magic + CRC)
    App->>FW: "halt off" command
    FW->>FW: Resume telemetry with new calibration

    Note over Clinician,Flash: On next boot...
    Flash->>FW: Reload saved coefficients (CRC-validated)
```

---

## 14. Development Timeline

```mermaid
flowchart LR
    subgraph FW_TIMELINE["Firmware Development"]
        direction LR
        F1["06 Aug\nRound 1\nBoot fix"]
        F2["06 Aug\nRound 2\n100 Hz stream"]
        F3["09 Aug\nRobustness\nDouble-buffer"]
        F4["11 Aug\nADC Fixed\nDMNGT=10"]
        F5["15 Aug\nMODE_REAL\nLive at boot"]
        F6["07-08 Sep\nCrypto +\nFaults"]
    end

    F1 --> F2 --> F3 --> F4 --> F5 --> F6

    subgraph APP_TIMELINE["Desktop App Releases"]
        direction LR
        A1["24 Jul\nv1.0.0\nInitial"]
        A2["07 Sep\nv1.1.0\nCrypto UI"]
        A3["10 Sep\nv1.2.0\nAuto-setup"]
        A4["10 Sep\nv1.2.1\nResync fix"]
        A5["Current\nv1.3.0\nScope + Config"]
    end

    A1 --> A2 --> A3 --> A4 --> A5
```

---

## 15. Progress Status

| Component | Feature | Status | Verification |
|-----------|---------|--------|-------------|
| Firmware | ADC / DMA / FIR acquisition | ✅ Complete | ✅ HW-verified |
| Firmware | 100 Hz telemetry frame | ✅ Complete | ✅ HW-verified |
| Firmware | 6-phase gait state machine | ✅ Complete | ⏳ Bench pending |
| Firmware | Faults / thresholds / coefficients | ✅ Complete | ⚠️ Simulated + partial |
| Firmware | AES-256-GCM encryption | ✅ Complete | ⚠️ Host vectors pass; bench pending |
| Firmware | Boot hardening | ✅ Complete | ✅ HW-verified |
| App | Transports (USB / BLE / Demo) | ✅ Complete | ✅ Functional |
| App | GUI (16 tabs) + charts | ✅ Complete | ✅ Captured / reviewed |
| App | Recording & export | ✅ Complete | ✅ Functional |
| App | Crypto session UI | ✅ Complete | ✅ Resync fix shipped |

---

## 16. Key Findings & Risks

> [!WARNING]
> **Documentation Drift:** Some documentation doesn't match the actual code:
> - Docs say USART2/PD5/PD6 for slave link → Code uses **USART3/PD8/PD9**
> - Docs claim 16 MHz ADC clock → Code correctly uses **4 MHz**
> - App docs show fault as uint8 → Code correctly uses **uint16** at offset 14
> - App battery defaults (10.5–13 V) don't match firmware's **4.2 V scale**

> [!CAUTION]
> - No CI pipeline or automated unit tests — verification is manual
> - End-to-end crypto bench test still pending

---

## 17. Recommended Next Steps

1. **Reconcile documentation** with the actual code constants
2. **Fix battery threshold defaults** in the app to match firmware's voltage scale
3. **Add automated test harness** for protocol parsing, CRC, state machine, and faults
4. **Add firmware CI** — compile + static analysis on every change
5. **Consolidate protocol constants** — single source of truth (`constants.js` in app)
6. **Complete crypto bench test** — end-to-end encrypted session on real hardware

---

## 18. Key Source Files Reference

### Firmware (`Core/Src/`)

| File | Purpose |
|------|---------|
| `main.c` | Boot, clock init, CLI command parser, UART callbacks |
| `adc.c` | ADC1 HAL init, 6-channel scan configuration |
| `adc_manager.c` | Register-level DMA setup, FIR filter, voltage scaling |
| `state_machine.c` | 6-phase gait detection with configurable bands |
| `smart_protocol.c` | 25-byte frame builder with CRC-16/MODBUS |
| `slave_comm.c` | USART3 slave link — 7-byte frames, median filter |
| `crypto.c` | AES-256-GCM with HW acceleration + SW fallback |
| `thresholds.c` | Threshold packet handler + flash persistence |
| `coefficients.c` | Calibration packet handler + flash persistence |
| `app_freertos.c` | RTOS objects, 100 Hz timer, telemetry callback |
| `sensor.c` | Per-channel stuck/flatline diagnostics |
| `fir_filter.c` | 51-tap Hamming low-pass filter implementation |

### Desktop App (`src/`)

| File | Purpose |
|------|---------|
| `main.js` | Electron main process — window, serial IPC, permissions |
| `preload.js` | Secure contextBridge API (no Node in renderer) |
| `renderer.js` | UI logic — tab management, chart updates, parser |
| `index.html` | Full HTML UI — all 16 tabs, Plotly integration |
| `scripts/constants.js` | Protocol constants shared across the app |

---

## 19. Glossary

| Term | Meaning |
|------|---------|
| **ADC** | Analog-to-Digital Converter — turns sensor voltages into numbers |
| **DMA** | Direct Memory Access — transfers data without using the CPU |
| **FIR** | Finite Impulse Response — a type of digital noise filter |
| **FreeRTOS** | Real-Time Operating System — manages tasks and timing |
| **CRC-16/MODBUS** | Error-checking code to detect data corruption |
| **AES-256-GCM** | Military-grade encryption algorithm |
| **PSK** | Pre-Shared Key — a secret password both sides know |
| **BLE / NUS** | Bluetooth Low Energy / Nordic UART Service |
| **Gait Cycle** | One complete step (heel strike → toe off → swing → next heel strike) |
| **Hysteresis** | Keeping the previous state when the signal is between two bands (prevents flickering) |
