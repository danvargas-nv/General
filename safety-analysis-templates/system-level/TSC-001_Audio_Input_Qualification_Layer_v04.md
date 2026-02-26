# TSC-001: Audio Input Qualification Layer (AIQL)

## 1. Header & Metadata

| Field | Value |
|-------|-------|
| **Document ID** | TSC-001 |
| **Title** | Audio Input Qualification Layer (AIQL) — Technical Safety Concept |
| **ASIL Target** | ASIL B |
| **Safety Goal** | SG-EMV-01 |
| **HARA Reference** | Audio_HARA / UC-React-to-EMV — Row: "Yield to Activated EMV (Highway), siren+lights, slow/stationary in front" |
| **Item** | AMO (Alpamayo) — Audio Input Path (on DRIVE AGX Thor / Hyperion 10) |
| **Version** | 0.9 |
| **Status** | In Development |
| **Author** | Safety Engineering |
| **Created** | 2026-02-22 |
| **Last Modified** | 2026-02-26 |
| **Classification** | Confidential |

### Document Traceability

| Relationship | Document |
|-------------|----------|
| Parent | Audio_HARA — Hazard Analysis and Risk Assessment (UC-React-to-EMV) |
| Sibling | FMEA-001 — System FMEA |
| Sibling | SOTIF-001 — SOTIF Analysis |
| Reference | REF-001 — ASIL Determination Matrix |
| Reference | REF-002 — Severity, Exposure, Controllability Scales |

### Revision History

| Version | Date | Author | Change Description |
|---------|------|--------|--------------------|
| 0.1 | 2026-02-22 | Safety Engineering | Initial draft — all 15 sections |
| 0.2 | 2026-02-22 | Safety Engineering | Added TSR-AIQL-014 (Single-Ended Plausibility Check); updated FM-05/FM-06 coverage; added FI-10; updated traceability |
| 0.3 | 2026-02-22 | Safety Engineering | Scoped AIQL to audio I/O qualification only; audio designated as secondary safety sensor (primary: camera/lidar/radar); removed TSR-AIQL-006 (cross-modal plausibility) and TSR-AIQL-014 (single-ended plausibility) — classifier correctness addressed at system level by Alpamayo multi-sensor fusion; removed camera interface; simplified safe state |
| 0.4 | 2026-02-23 | Safety Engineering | Confirmed in-cabin microphone placement; added Speaker-Microphone Loopback BIST (Built-In Self-Test) with MRM transition; added TSR-AIQL-015 through -019; added FM-09/FM-10/FM-11; added AoU-011/AoU-012; added FI-10 through FI-14; added TC-AUDIO-05; added Appendix E (BIST architecture); updated traceability and coverage |
| 0.5 | 2026-02-23 | Safety Engineering | Corrected platform architecture framing: Alpamayo is a foundation-model research layer above NDAS/DRIVE AV, not the direct runtime consumer. Replaced all "Alpamayo" runtime references with "NDAS DRIVE AV" (the on-vehicle perception pipeline on DRIVE AGX Thor / Hyperion 10). Updated Item field, boundary diagrams, safety goal, failure mode effects, TSRs, state machine, FFI analysis, V&V, SOTIF, traceability, open items, and glossary. Added NDAS, DRIVE AGX Thor, DRIVE Hyperion 10 to glossary. |
| 0.6 | 2026-02-23 | Safety Engineering | Added SysML architecture views: Block Definition Diagram (4.1.1), Internal Block Diagram (4.1.2), Interface Block Definitions (4.2.6), Block/Part Summary Table (4.4), and Data Flow Sequences for nominal processing, startup BIST, and periodic BIST (4.5). |
| 0.7 | 2026-02-24 | Safety Engineering | Aligned to Audio_HARA.xlsx (UC-React-to-EMV): corrected S/E/C from S2/E3/C2 to S3/E2/C3; updated hazardous event to "collision with EMV slow/stationary in front on highway"; renamed SG-03 to SG-EMV-01; updated safety goal text; added full HARA scenario context table (Section 5.1); updated FTTI justification for highway EMV-in-front scenario; updated SOTIF TC-AUDIO-01 hazardous behavior; partially resolved OI-02; updated HARA references throughout. |
| 0.8 | 2026-02-25 | Safety Engineering | Corrected architecture framing: AMO (Alpamayo) is the direct on-vehicle consumer of microphone input, not "NDAS DRIVE AV" generically. ASIL B derives from FFI requirements within AMO — QM mic data must be qualified before entering AMO. Replaced "NDAS DRIVE AV" with "AMO" throughout where referring to the direct audio consumer (perception, fusion, sensor processing). Retained "NDAS DRIVE AV" for broader AD stack references (planning, control, Safety Force Field). Updated platform context, boundary diagrams, BDD, data flow diagrams, failure mode effects, TSRs, state machine, FFI analysis, SOTIF, traceability, open items, and glossary. Added AMO glossary entry. |
| 0.9 | 2026-02-26 | Safety Engineering | Major architecture revision: (1) Exterior microphones replace in-cabin microphones per systems engineering input; (2) AIQL monitor placed on Thor Main (BP SW), receiving audio via A2B, reporting to THOR FSI via SOC_Error/FSI_I2C; (3) Speaker-mic loopback BIST reclassified as optional/customer-dependent (no exterior speakers on Hyperion reference platform); (4) Added passive qualification strategy as primary approach: noise profile monitoring (TSR-AIQL-020), cross-mic plausibility (TSR-AIQL-021), input sanitization (TSR-AIQL-022); (5) Added FM-12 (mic blockage), FM-13 (cross-mic failure); (6) Updated AoU-009 to exterior mic specs, added AoU-013 (A2B frame structure), AoU-014 (mic quantity); (7) Updated BDD, IBD, boundary diagrams for BP/A2B/THOR FSI topology; (8) Reframed SOTIF TC-AUDIO-04/05 for exterior mics; (9) Added FI-15/16/17 for passive check fault injection; (10) Added OI-15 through OI-18 for A2B confirmation, FSI path, mic quantity, NDAS EVD commitment. |

---

## 2. Scope & Purpose

### 2.1 Scope

This Technical Safety Concept (TSC) defines the **Audio Input Qualification Layer (AIQL)** — an ASIL B software component running on **Thor Main (Base Platform SW)** at the boundary between the QM-rated audio subsystem and the ASIL-rated AMO perception pipeline. The AIQL qualifies exterior microphone data received via the **Automotive Audio Bus (A2B)** before it reaches AMO.

**Platform context**: The AIQL runs on Thor Main as part of the Base Platform (BP) SW. Exterior microphones (QM) connect to the BP board via A2B (up to 16 channels supported on Maarva). The AIQL processes audio frames from the A2B path and outputs qualified audio to AMO (Alpamayo), the on-vehicle perception system. Safety signaling (error reporting, MRM requests) is communicated to **THOR FSI** via SOC_Error / FSI_I2C interfaces. AMO is the **direct consumer of microphone input** — it fuses qualified audio with camera, LiDAR, radar, and ultrasonic data for autonomous driving perception. The ASIL B requirement on the AIQL derives from Freedom From Interference (FFI) requirements within AMO: QM microphone data must be qualified before entering the ASIL-rated AMO perception pipeline. AMO feeds its perception outputs into the broader NDAS DRIVE AV stack (planning, control, Safety Force Field).

Audio is designated as a **secondary safety sensor**. The primary sensors for emergency vehicle detection and all driving decisions are camera, LiDAR, and radar. Audio provides supplementary siren/horn detection that enhances overall detection confidence but is not the sole or primary input for any safety-critical decision. AMO multi-sensor fusion architecture is responsible for combining all sensor inputs and determining the appropriate driving response.

The AIQL qualifies QM audio input at the I/O boundary by implementing safety mechanisms that detect, contain, and mitigate data integrity failure modes in the audio input path. This follows the element-out-of-context (SEooC) integration approach per ISO 26262-8 Clause 12.

#### In Scope

- Qualification of the audio data stream at the AMO input boundary
- Freshness, integrity, sequence, and range validation of audio frames
- Noise profile monitoring and input sanitization
- Cross-microphone plausibility checking (when multiple mics are available)
- Safe state definition and graceful degradation strategy
- Temporal and spatial Freedom From Interference (FFI) mechanisms
- Fault injection verification campaign for the AIQL
- ASIL decomposition argument per ISO 26262-9
- Acoustic loopback via exterior sound source (optional, customer-dependent enhancement)

#### Out of Scope

- Design or modification of the QM audio hardware (exterior microphones, codec, amplifiers)
- Design or modification of the QM audio driver or DSP firmware
- Customer-side audio front-end hardware selection or diagnostics (codec, A2B transceiver)
- The siren/horn classifier algorithm itself (treated as QM element with AoUs)
- Classifier correctness validation (semantic accuracy of siren/horn detection is a system-level concern addressed by AMO multi-sensor fusion with primary sensors)
- Cross-modal plausibility checking with camera or other sensors (sensor fusion is AMO's responsibility)
- AMO internals beyond the audio input interface
- Alpamayo foundation model training, distillation, or cloud infrastructure
- Non-safety audio functions (in-cabin entertainment, voice commands)
- Cybersecurity requirements for the audio path (addressed in separate cybersecurity concept)
- Customer-specific exterior speaker availability or acoustic loopback implementation (defined as optional enhancement)

### 2.2 Purpose

AMO (Alpamayo) — the on-vehicle perception system running on DRIVE AGX Thor within the Hyperion 10 reference architecture — directly consumes exterior microphone input for siren/horn detection that influences safety-critical driving decisions, specifically yielding to emergency vehicles. AMO's perception outputs feed into the broader NDAS DRIVE AV stack for planning and control. The audio hardware and software (exterior microphones, codec, A2B transceiver, DSP, driver, classifier) are developed to QM (Quality Management) level.

Without qualification, unqualified audio input creates Freedom From Interference (FFI) issues within AMO. Specifically:

1. **Corrupted audio data** could inject erroneous siren/horn signals into AMO's fusion layer, degrading fusion confidence
2. **Missing or stale audio data** could cause AMO to operate with outdated supplementary information without awareness of its invalidity
3. **Grossly off signals** (blocked microphone, environmental damage, wiring fault) could degrade AMO perception without detection

The AIQL resolves these FFI issues by implementing ASIL B safety mechanisms on Thor Main (BP SW) at the I/O boundary, qualifying the QM audio input received via A2B before it enters the AMO perception pipeline. The AIQL ensures that audio data entering AMO is fresh, intact, and within expected ranges — or explicitly marked as not qualified so the fusion layer can exclude it. This avoids the prohibitively expensive alternative of developing the entire audio hardware and software stack to ASIL B.

The AIQL implements a **layered passive qualification strategy** that does not depend on customer-side audio hardware capabilities:

1. **Noise profile monitoring**: Verify that incoming audio has characteristics consistent with a functioning exterior microphone (ambient noise present, no DC rail, no persistent saturation)
2. **Cross-microphone plausibility**: Compare channels across the mic array; flag any channel that deviates significantly from its peers (requires minimum 2 microphones)
3. **Input sanitization**: Verify amplitude range, noise floor, absence of persistent clipping or DC offset before forwarding to AMO
4. **Data integrity checks**: Freshness, alive counter, CRC, sequence counter, and range validation on every A2B frame
5. **Acoustic loopback** (optional, customer-dependent): If an exterior sound source is available on the vehicle (e.g., backup warning speaker, pedestrian alerting speaker), the AIQL can execute a coded chirp stimulus and verify the received signal against a reference profile for end-to-end acoustic path verification

**Note**: Classifier correctness (whether the siren/horn classifier accurately identifies emergency vehicle audio) is a system-level concern. Because audio is a secondary sensor, misclassification is mitigated by AMO multi-sensor fusion with primary sensors (camera, LiDAR, radar), not by the AIQL.

### 2.3 Boundary Definition

```
                 Customer Side (QM)                  NVIDIA BP (ASIL B)        NVIDIA (ASIL)
                 |                                   |                         |
  [Exterior  ] ---> [Codec] ---> [A2B Bus] -------> [AIQL Monitor] --------> [AMO]
  [Microphones]     |(ADC)  |                        (Thor Main,              (Alpamayo)
                    |                                 BP SW)                   |
     QM HW          QM HW        QM Bus        ASIL B SW               ASIL (perception)
                                                     |
                                                     | safety signaling
                                                     v
                                                [THOR FSI]
                                                (SOC_Error / FSI_I2C)

  Optional (customer-dependent):
  [Exterior   ] <-- [Codec] <-- [Acoustic Loopback ] <-+
  [Speaker    ]     |(DAC)  |   | Stimulus (AIQL)   |   |
  (backup/AVAS)                                         |
     QM HW          QM HW       ASIL B SW              |
                                                        |
                    Loopback: Speaker output ---------> Microphone input
                              (verified by AIQL correlation match)
```

The AIQL sits on Thor Main (BP SW) at the exact boundary between the QM audio subsystem (A2B input) and the ASIL-rated AMO perception pipeline. All audio data entering AMO passes through the AIQL — there is no bypass path. The A2B bus and AIQL are both on the Base Platform board, confirming the data path is accessible to BP SW.

---

## 3. Referenced Standards

### 3.1 Normative References

| Standard | Part | Title | Applicability |
|----------|------|-------|---------------|
| ISO 26262:2018 | Part 1 | Vocabulary | Terminology used throughout |
| ISO 26262:2018 | Part 3 | Concept phase | HARA, safety goals, ASIL determination |
| ISO 26262:2018 | Part 4 | Product development at the system level | TSC derivation, technical safety requirements |
| ISO 26262:2018 | Part 5 | Product development at the hardware level | HW safety mechanisms, SPFM/LFM targets |
| ISO 26262:2018 | Part 6 | Product development at the software level | SW safety requirements, architecture, verification |
| ISO 26262:2018 | Part 8 | Supporting processes | Clause 12: Interfaces within distributed developments (SEooC integration) |
| ISO 26262:2018 | Part 9 | ASIL-oriented and safety-oriented analyses | ASIL decomposition, dependent failure analysis, FFI |
| ISO 21448:2022 | — | Safety of the intended functionality (SOTIF) | Triggering conditions for audio misclassification |
| AUTOSAR | E2E Profile 1 | End-to-end communication protection | CRC-32, alive counter, sequence counter specification |

### 3.2 Informative References

| Reference | Description |
|-----------|-------------|
| REF-001 | ASIL Determination Matrix (project-internal) |
| REF-002 | Severity, Exposure, Controllability Scales (project-internal) |
| NFPA 1901 | Standard for Automotive Fire Apparatus — siren specifications |
| SAE J1772 | Electric vehicle emergency sound requirements |
| Federal Siren Standard (USA) | Title 49 CFR Part 571.141 — minimum sound levels |

---

## 4. System Architecture

### 4.1 Audio Input Path — Data Flow

```
  Customer Side (QM)                           NVIDIA Base Platform
  ==================                           ====================

+==============+     +============+     +=========+     +====================+
|  Exterior    |---->|   Codec    |---->| A2B Bus |---->| Thor Main SoC      |
|  Microphone  |     | (ADC)      |     |         |     |                    |
|  Array       |     +============+     +=========+     |  +==============+  |
+==============+          |                   |         |  | AIQL Monitor |  |
                          |                   |         |  | (ASIL B,     |  |
   Analog audio      Digital audio       A2B frames     |  |  BP SW)      |  |
   (electrical)      (to A2B)            (digital)      |  +======+=======+  |
                          |                   |         |         |          |
   +--- QM HW -----------+---- QM HW --------+         |         |          |
                                                        |    +----+-----+    |
                                                        |    |          |    |
                                                        |    v          v    |
                                                        | [AMO]   [THOR FSI]|
                                                        | (ASIL)  (safety   |
                                                        |          mgmt)    |
                                                        +====================+

              +====================================================+
              | AIQL Internal Architecture (Thor Main, BP SW)      |
              |                                                    |
              | +-------------+  +---------------+  +------------+ |
              | | Freshness & |  | Integrity &   |  | Noise      | |
              | | Alive Check |  | Range Check   |  | Profile &  | |
              | |             |  |               |  | Plausibil. | |
              | | - Timestamp |  | - CRC-32      |  |            | |
              | | - Alive Ctr |  | - Seq Counter |  | - Noise    | |
              | | - Freshness |  | - Range Valid |  |   Floor    | |
              | +------+------+  +-------+-------+  | - Cross-Mic| |
              |        |                 |           | - Sanitize | |
              |        |                 |           +------+-----+ |
              |        v                 v                  v       |
              | +---------------------------------------------------+
              | | Qualification Decision & State Machine              |
              | |                                                    |
              | | QUALIFIED ---> DEGRADED ---> NOT_QUALIFIED         |
              | |                                 |                  |
              | |                                 +--> MRM_REQUESTED |
              | +---------------------------------------------------+
              |        |                                   |        |
              |        v                                   v        |
              |  qualified_audio_frame              safety status   |
              |  + qualification_status             to THOR FSI     |
              |  to AMO                             (SOC_Error)     |
              +====================================================+

  Optional (customer-dependent):
  +==============+     +============+
  |  Exterior    |<----|   Codec    |<---- Acoustic Loopback Stimulus
  |  Speaker     |     | (DAC)      |      (from AIQL, if speaker available)
  |  (QM HW)    |     +============+
  +==============+
        |
        +-- acoustic coupling --> Exterior Microphone(s)
                                  (verified by AIQL correlation match)
```

#### 4.1.1 System Context — Block Definition Diagram (SysML BDD)

The following diagram defines the block hierarchy and associations for the AIQL within the Base Platform on DRIVE AGX Thor. Map each box to a SysML `«block»` with the indicated stereotype and ASIL classification.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│ «block» DRIVE_AGX_Thor_Platform                                    Hyperion 10  │
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐   │
│  │                  Customer Side — QM Boundary                              │   │
│  │                                                                           │   │
│  │  ┌──────────────┐     ┌──────────────┐     ┌──────────────────────────┐  │   │
│  │  │ «block»      │     │ «block»      │     │ «block»                  │  │   │
│  │  │ ExteriorMic  │     │ AudioCodec   │     │ A2B_Transceiver          │  │   │
│  │  │ Array        │     │              │     │                          │  │   │
│  │  │──────────────│     │──────────────│     │──────────────────────────│  │   │
│  │  │ «QM HW»     │     │ «QM HW»     │     │ «QM HW»                 │  │   │
│  │  │ ASIL: QM    │     │ ASIL: QM    │     │ ASIL: QM               │  │   │
│  │  │──────────────│     │──────────────│     │──────────────────────────│  │   │
│  │  │ numChannels  │     │ adcMode:     │     │ maxChannels: 16         │  │   │
│  │  │  : TBD      │     │  vendor-     │     │ busType: A2B            │  │   │
│  │  │ sampleRate:  │     │  specific   │     │ frameRate: 20 Hz        │  │   │
│  │  │  48000 Hz   │     │ sampleBits:  │     │──────────────────────────│  │   │
│  │  │ freqRange:   │     │  16         │     │ +packFrame()            │  │   │
│  │  │  20-20kHz   │     │──────────────│     │ +writeCRC32()           │  │   │
│  │  │ placement:   │     │ +adcConvert()│     │ +incrAliveCounter()    │  │   │
│  │  │  exterior   │     └──────┬───────┘     └────────────┬───────────┘  │   │
│  │  └──────┬───────┘            │                          │              │   │
│  │         │ analog             │ digital                  │ A2B frames   │   │
│  │         │ audio              │ audio                    │              │   │
│  │         ▼                    ▼                          │              │   │
│  │         ├────────────────────┘                          │              │   │
│  │         │                                               │              │   │
│  │  ┌──────┴───────┐                                      │              │   │
│  │  │ «block»      │                                      │              │   │
│  │  │ SirenHorn    │                                      │              │   │
│  │  │ Classifier   │                                      │              │   │
│  │  │──────────────│                                      │              │   │
│  │  │ «QM SW»     │                                      │              │   │
│  │  │ ASIL: QM    │                                      │              │   │
│  │  │──────────────│                                      │              │   │
│  │  │ +classify()  │                                      │              │   │
│  │  │  siren_p     │                                      │              │   │
│  │  │  horn_p      │                                      │              │   │
│  │  │  direction   │                                      │              │   │
│  │  │  confidence  │                                      │              │   │
│  │  └──────────────┘                                      │              │   │
│  │                                                        │              │   │
│  │  ┌──────────────┐  (optional, customer-dependent)      │              │   │
│  │  │ «block»      │                                      │              │   │
│  │  │ ExteriorSpkr │  Acoustic loopback source            │              │   │
│  │  │──────────────│  (backup warning, AVAS, etc.)        │              │   │
│  │  │ «QM HW»     │                                      │              │   │
│  │  │ ASIL: QM    │                                      │              │   │
│  │  └──────────────┘                                      │              │   │
│  └────────────────────────────────────────────────────────┼──────────────┘   │
│                                                           │                  │
│                                                           │ A2B audioFrame   │
│                                         ┌─────────────────┼──────────────┐   │
│  ┌───────────────────┐                  │ ASIL B Boundary  │              │   │
│  │ «block»           │                  │ (Thor Main,      ▼              │   │
│  │ THOR_FSI          │◄──safetyStatus───│  BP SW)                        │   │
│  │───────────────────│  (SOC_Error /    │  ┌─────────────────────────┐  │   │
│  │ «ASIL D»         │   FSI_I2C)       │  │ «block»                 │  │   │
│  │───────────────────│                  │  │ AIQL                    │  │   │
│  │ +monitorWatchdog()│                  │  │─────────────────────────│  │   │
│  │ +handleMRM()      │                  │  │ «ASIL B SW»            │  │   │
│  │ +reportSOC_Error()│                  │  │─────────────────────────│  │   │
│  └───────────────────┘                  │  │ (see IBD in 4.1.2)     │  │   │
│                                         │  └────────────┬────────────┘  │   │
│                                         │               │               │   │
│                                         └───────────────┼───────────────┘   │
│                                                         │                    │
│                                                         │ qualifiedFrame     │
│                                                         ▼                    │
│                          ┌────────────────────────────────────────────────┐  │
│                          │ «block» AMO (Alpamayo)                          │  │
│                          │────────────────────────────────────────────────│  │
│                          │ «ASIL (mixed)» — on-vehicle perception         │  │
│                          │────────────────────────────────────────────────│  │
│                          │ +multiSensorFusion(camera, lidar, radar,      │  │
│                          │                    audio, ultrasonic)          │  │
│                          │ +emergencyVehicleDetection()                   │  │
│                          │ +feedNDAS_DRIVE_AV(planning, control, SFF)    │  │
│                          └────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ «block» DRIVE_OS  (DriveOS 7, ASIL-D)                                 │  │
│  │ CUDA · TensorRT · NvMedia · Hardware Hypervisor · MIG Partitioning    │  │
│  │ MPU/MMU · Watchdog Timer · DMA Firewall · Time-Partitioned Scheduler  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 AIQL Internal Structure — Internal Block Diagram (SysML IBD)

The following diagram defines the internal parts, connectors, and item flows within the AIQL block. Each sub-block maps to a SysML `«part»` typed by the corresponding block definition.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│ ibd [block] AIQL                                                          ASIL B      │
│                                                                                        │
│  PORTS (SysML FlowPorts):                                                             │
│    pIn_rawFrame     : in  RawAudioFrame       (from A2B, via Thor Main)              │
│    pOut_qualFrame   : out QualifiedAudioFrame  (to AMO)                               │
│    pOut_safetyStatus: out SafetyStatus         (to THOR FSI, via SOC_Error/FSI_I2C)  │
│    pOut_bistSignal  : out BISTSignal           (optional: to Codec DAC -> Speaker)    │
│    pOut_diagLog     : out DiagnosticEvent      (to NV storage)                        │
│                                                                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                                  │  │
│  │                      ┌──────────────────────────────────┐                        │  │
│  │  pIn_rawFrame ──────>│ «part» inputDemux                │                        │  │
│  │                      │ Demultiplexes incoming frame      │                        │  │
│  │                      │ fields for parallel checks        │                        │  │
│  │                      └──┬───────┬───────┬───────┬─────┬─┘                        │  │
│  │                         │       │       │       │     │                           │  │
│  │              timestamp  │ alive │ crc32 │ seq   │ audio_data                      │  │
│  │              _us        │ _ctr  │       │ _ctr  │ + classifier                    │  │
│  │                         │       │       │       │                                 │  │
│  │                    ┌────▼────┐  │  ┌────▼────┐  │  ┌────────────────────────┐     │  │
│  │                    │ «part» │  │  │ «part» │  │  │ «part»                │     │  │
│  │                    │ fresh  │  │  │ crc    │  │  │ rangeValidator        │     │  │
│  │                    │ Check  │  │  │ Check  │  │  │                      │     │  │
│  │                    │────────│  │  │────────│  │  │──────────────────────│     │  │
│  │                    │stale   │  │  │compute │  │  │ sampleAmplitude:    │     │  │
│  │                    │Thresh: │  │  │CRC over│  │  │  saturation < 5%   │     │  │
│  │                    │ 100 ms │  │  │all flds│  │  │ dcOffset:           │     │  │
│  │                    │        │  │  │compare │  │  │  [-500, +500] LSB  │     │  │
│  │                    │out:    │  │  │to rxd  │  │  │ noiseFloorRMS:      │     │  │
│  │                    │ PASS/  │  │  │        │  │  │  > 10 LSB          │     │  │
│  │                    │ FAIL + │  │  │out:    │  │  │ classifierProb:     │     │  │
│  │                    │ FF_    │  │  │ PASS/  │  │  │  [0.0, 1.0]        │     │  │
│  │                    │ FRESH  │  │  │ FAIL + │  │  │ directionDOA:       │     │  │
│  │                    └───┬────┘  │  │ FF_CRC │  │  │  [-180, +180]      │     │  │
│  │                        │       │  └───┬────┘  │  │                      │     │  │
│  │                        │  ┌────▼────┐ │       │  │out: PASS/FAIL +     │     │  │
│  │                        │  │ «part» │ │  ┌────▼─┤    FF_RANGE          │     │  │
│  │                        │  │ alive  │ │  │     └──────────┬───────────┘     │  │
│  │                        │  │ Check  │ │  │                │                 │  │
│  │                        │  │────────│ │  │                │                 │  │
│  │                        │  │prev +1 │ │  │                │                 │  │
│  │                        │  │mod 256 │ │  │                │                 │  │
│  │                        │  │3 consec│ │  │                │                 │  │
│  │                        │  │violat  │ │  │                │                 │  │
│  │                        │  │->DEGRDD│ │  │                │                 │  │
│  │                        │  │        │ │  │                │                 │  │
│  │                        │  │out:    │ │  ┌────▼────┐      │                 │  │
│  │                        │  │ PASS/  │ │  │ «part» │      │                 │  │
│  │                        │  │ FAIL + │ │  │ seq    │      │                 │  │
│  │                        │  │ FF_    │ │  │ Check  │      │                 │  │
│  │                        │  │ ALIVE  │ │  │────────│      │                 │  │
│  │                        │  └───┬────┘ │  │monoton │      │                 │  │
│  │                        │      │      │  │incr w/ │      │                 │  │
│  │                        │      │      │  │16-bit  │      │                 │  │
│  │                        │      │      │  │wrap    │      │                 │  │
│  │                        │      │      │  │        │      │                 │  │
│  │                        │      │      │  │out:    │      │                 │  │
│  │                        │      │      │  │ PASS/  │      │                 │  │
│  │                        │      │      │  │ FAIL + │      │                 │  │
│  │                        │      │      │  │ FF_SEQ │      │                 │  │
│  │                        │      │      │  └───┬────┘      │                 │  │
│  │                        │      │      │      │           │                 │  │
│  │   ─────────────────────┴──────┴──────┴──────┴───────────┘                 │  │
│  │   │  checkResults[5] : {PASS|FAIL, failure_flag}                          │  │
│  │   │                                                                       │  │
│  │   ▼                                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ «part» qualificationDecision                                       │   │  │
│  │  │ (Qualification Decision Engine)                                    │   │  │
│  │  │────────────────────────────────────────────────────────────────────│   │  │
│  │  │                                                                    │   │  │
│  │  │  Inputs:                                                           │   │  │
│  │  │    checkResults[5]  <- freshnessCheck, aliveCheck, crcCheck,      │   │  │
│  │  │                       seqCheck, rangeValidator                    │   │  │
│  │  │    bistResult       <- bistModule.passFailResult                  │   │  │
│  │  │    currentState     <- stateMachine.state                         │   │  │
│  │  │                                                                    │   │  │
│  │  │  Logic:                                                            │   │  │
│  │  │    allChecksPassed = AND(checkResults[0..4])                       │   │  │
│  │  │    anyCheckFailed  = OR(NOT(checkResults[0..4]))                   │   │  │
│  │  │    bistFailed      = bistResult.failed AND retriesExhausted       │   │  │
│  │  │                                                                    │   │  │
│  │  │  Output:                                                           │   │  │
│  │  │    stateTransitionCmd  -> stateMachine                            │   │  │
│  │  │    aggregateFlags      -> failure_flags : uint16                  │   │  │
│  │  │                                                                    │   │  │
│  │  └──────────────────────────────────┬─────────────────────────────────┘   │  │
│  │                                     │                                     │  │
│  │                 stateTransitionCmd   │   aggregateFlags                    │  │
│  │                                     ▼                                     │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ «part» stateMachine                                                │   │  │
│  │  │ (AIQL Qualification State Machine — see stm in Section 10.1)       │   │  │
│  │  │────────────────────────────────────────────────────────────────────│   │  │
│  │  │                                                                    │   │  │
│  │  │  ┌───────────┐  anyFail   ┌──────────┐  3 consec  ┌────────────┐ │   │  │
│  │  │  │ QUALIFIED ├───────────>│ DEGRADED ├──failures─>│NOT_QUALFIED│ │   │  │
│  │  │  │           │<───────────┤          │<───────────┤            │ │   │  │
│  │  │  │           │  10 valid  │          │  20 valid  │            │ │   │  │
│  │  │  └───────────┘  frames    └──────────┘  frames    └─────┬──────┘ │   │  │
│  │  │       ▲                                          BIST   │        │   │  │
│  │  │       │ startup                                  fail   │        │   │  │
│  │  │       │ BIST pass                              (retries │        │   │  │
│  │  │  ┌────┴──────┐                               exhausted)│        │   │  │
│  │  │  │   INIT    │                                         ▼        │   │  │
│  │  │  │ (power-on)│                              ┌──────────────┐    │   │  │
│  │  │  └───────────┘                              │MRM_REQUESTED │    │   │  │
│  │  │                                             │  (terminal)  │    │   │  │
│  │  │  Output:                                    └──────────────┘    │   │  │
│  │  │    state              : enum {QUALIFIED, DEGRADED,              │   │  │
│  │  │                               NOT_QUALIFIED, MRM_REQUESTED}     │   │  │
│  │  │    qualification_status: enum8 (0x01, 0x02, 0x03)               │   │  │
│  │  │    mrmRequestTrigger  : bool                                    │   │  │
│  │  └──────────────┬──────────────────────────┬────────────────┬─────┘   │  │
│  │                 │                          │                │          │  │
│  │                 │ qualification_status      │ mrmRequest     │ state    │  │
│  │                 │ + failure_flags           │ Trigger        │          │  │
│  │                 ▼                          ▼                │          │  │
│  │  ┌──────────────────────────┐   ┌─────────────────┐        │          │  │
│  │  │ «part» outputAssembler   │   │                 │        │          │  │
│  │  │                          │   │  pOut_mrmReq ───┼──────> pOut_      │  │
│  │  │ Assembles output frame:  │   │                 │        mrmRequest │  │
│  │  │  if QUALIFIED:           │   └─────────────────┘                   │  │
│  │  │   pass-through audio +   │                                         │  │
│  │  │   classifier             │                                         │  │
│  │  │  if DEGRADED:            │                                         │  │
│  │  │   pass-through +         │                                         │  │
│  │  │   failure_flags          │                                         │  │
│  │  │  if NOT_QUALIFIED:       │                                         │  │
│  │  │   zero-fill audio +      │                                         │  │
│  │  │   zero-fill classifier   │                                         │  │
│  │  │                          │                                         │  │
│  │  │  Compute aiql_crc32      │                                         │  │
│  │  │  Output at 20 Hz always  │                                         │  │
│  │  └─────────────┬────────────┘                                         │  │
│  │                │                                                      │  │
│  │                │ QualifiedAudioFrame                                   │  │
│  │                ▼                                                      │  │
│  │           pOut_qualFrame ──────────────────────> (to AMO)   │  │
│  │                                                                       │  │
│  │  ═════════════════════════════════════════════════════════════════     │  │
│  │  Noise Profile & Plausibility Subsystem (Primary Diagnostic)          │  │
│  │  ═════════════════════════════════════════════════════════════════     │  │
│  │                                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────┐   │  │
│  │  │ «part» passiveMonitor                                          │   │  │
│  │  │ (Passive Qualification Checks)                                 │   │  │
│  │  │────────────────────────────────────────────────────────────────│   │  │
│  │  │                                                                │   │  │
│  │  │  ┌──────────────────────┐   ┌──────────────────────────────┐  │   │  │
│  │  │  │ «part»               │   │ «part»                       │  │   │  │
│  │  │  │ noiseProfileCheck    │   │ crossMicPlausibility         │  │   │  │
│  │  │  │                      │   │                              │  │   │  │
│  │  │  │ Per-channel checks:  │   │ Multi-channel comparison:   │  │   │  │
│  │  │  │  silence: RMS < 10  │   │  RMS & spectral across all  │  │   │  │
│  │  │  │  DC rail: mean near │   │  exterior mic channels      │  │   │  │
│  │  │  │   +/- rail          │   │  threshold: X dB from       │  │   │  │
│  │  │  │  persistent sat:    │   │   median (default: 12 dB)   │  │   │  │
│  │  │  │   >5% clipped 3+   │   │  requires: >= 2 mics        │  │   │  │
│  │  │  │   frames            │   │                              │  │   │  │
│  │  │  │  spectral anomaly   │   │ out: per-channel PASS/FAIL  │  │   │  │
│  │  │  │                      │   │      + FF_CROSSMIC           │  │   │  │
│  │  │  │ out: per-channel     │   └──────────────┬───────────────┘  │   │  │
│  │  │  │  PASS/FAIL +         │                  │                  │   │  │
│  │  │  │  FF_NOISE_PROFILE    │                  │                  │   │  │
│  │  │  └──────────┬───────────┘                  │                  │   │  │
│  │  │             │           ┌───────────────────┘                  │   │  │
│  │  │             │           │                                      │   │  │
│  │  │             │           v                                      │   │  │
│  │  │             │  ┌──────────────────────────────┐                │   │  │
│  │  │             │  │ «part»                       │                │   │  │
│  │  │             │  │ inputSanitizer               │                │   │  │
│  │  │             │  │                              │                │   │  │
│  │  │             └─>│ Final safety gate before AMO │                │   │  │
│  │  │                │  amplitude, noise floor,     │                │   │  │
│  │  │                │  clipping, DC offset checks  │                │   │  │
│  │  │                │                              │                │   │  │
│  │  │                │ action: zero-fill + flag     │                │   │  │
│  │  │                │ non-conforming frames        │                │   │  │
│  │  │                │                              │                │   │  │
│  │  │                │ out: sanitized result        │                │   │  │
│  │  │                │   -> qualificationDecision   │                │   │  │
│  │  │                └──────────────────────────────┘                │   │  │
│  │  │                                                                │   │  │
│  │  └────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  │  ═════════════════════════════════════════════════════════════════     │  │
│  │  BIST Subsystem (Optional, Customer-Dependent)                        │  │
│  │  ═════════════════════════════════════════════════════════════════     │  │
│  │  Active only when an exterior speaker is available on the vehicle.    │  │
│  │  If no speaker (Hyperion reference), BIST is disabled and passive     │  │
│  │  monitoring above serves as the sole diagnostic approach.             │  │
│  │                                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────┐   │  │
│  │  │ «part» bistModule (optional)                                   │   │  │
│  │  │ (Speaker-Microphone Loopback BIST)                             │   │  │
│  │  │────────────────────────────────────────────────────────────────│   │  │
│  │  │  signalGenerator ──> spectralMatchVerifier ──> bistRetryCtrl  │   │  │
│  │  │  See Appendix E for full BIST architecture details.            │   │  │
│  │  │                                                                │   │  │
│  │  │  pOut_bistSignal ──────> (to Codec DAC -> Exterior Speaker)   │   │  │
│  │  │  out: passFailResult ──> qualificationDecision                │   │  │
│  │  └───────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  │  ┌────────────────────────────────────────────────────────────────┐   │  │
│  │  │ «part» hwProtection (provided by DRIVE OS)                     │   │  │
│  │  │────────────────────────────────────────────────────────────────│   │  │
│  │  │ mpuMmu      : spatial FFI  — isolates AIQL memory from QM     │   │  │
│  │  │ watchdog    : temporal FFI — 100 ms timeout                    │   │  │
│  │  │ dmaFirewall : restricts QM DMA to audio buffer region          │   │  │
│  │  │ scheduler   : guaranteed CPU budget (dedicated core or 10%)    │   │  │
│  │  └────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Interface Definitions

#### 4.2.1 Input Interface: Audio Driver → AIQL

| Field | Type | Size (bytes) | Description |
|-------|------|-------------|-------------|
| `frame_id` | uint32 | 4 | Monotonically increasing frame identifier |
| `timestamp_us` | uint64 | 8 | Capture timestamp in microseconds (system clock) |
| `alive_counter` | uint8 | 1 | Modulo-256 alive counter, incremented per frame |
| `sequence_counter` | uint16 | 2 | Monotonically increasing sequence number (wraps at 65535) |
| `sample_rate_hz` | uint32 | 4 | Audio sample rate (expected: 48000 Hz) |
| `num_channels` | uint8 | 1 | Number of audio channels (expected: 4 for mic array) |
| `num_samples` | uint16 | 2 | Number of samples per channel in this frame |
| `audio_data` | int16[] | variable | Interleaved PCM audio samples (16-bit signed) |
| `classifier_output` | struct | 16 | Siren/horn classifier result (see 4.2.3) |
| `crc32` | uint32 | 4 | CRC-32 over all preceding fields |

**Total header size**: 42 bytes (fixed) + audio_data (variable) + classifier (16 bytes)

#### 4.2.2 Output Interface: AIQL → AMO

| Field | Type | Size (bytes) | Description |
|-------|------|-------------|-------------|
| `frame_id` | uint32 | 4 | Original frame ID (pass-through) |
| `timestamp_us` | uint64 | 8 | Original capture timestamp (pass-through) |
| `qualification_status` | enum8 | 1 | QUALIFIED (0x01), DEGRADED (0x02), NOT_QUALIFIED (0x03) |
| `audio_data` | int16[] | variable | Original audio samples (pass-through when qualified) |
| `classifier_output` | struct | 16 | Original classifier output (pass-through when qualified) |
| `failure_flags` | uint16 | 2 | Bitfield of active failure detections (see 4.2.4) |
| `aiql_crc32` | uint32 | 4 | CRC-32 over all AIQL output fields |

#### 4.2.3 Classifier Output Structure

| Field | Type | Size (bytes) | Description |
|-------|------|-------------|-------------|
| `siren_probability` | float32 | 4 | Probability of siren presence [0.0, 1.0] |
| `horn_probability` | float32 | 4 | Probability of horn presence [0.0, 1.0] |
| `direction_deg` | float32 | 4 | Estimated direction of arrival in degrees [-180, +180] |
| `confidence` | float32 | 4 | Overall classifier confidence [0.0, 1.0] |

#### 4.2.4 Failure Flags Bitfield

| Bit | Flag | Description |
|-----|------|-------------|
| 0 | `FF_FRESHNESS` | Freshness check failed (stale data) |
| 1 | `FF_ALIVE` | Alive counter check failed |
| 2 | `FF_CRC` | CRC-32 integrity check failed |
| 3 | `FF_SEQUENCE` | Sequence counter check failed |
| 4 | `FF_RANGE` | Signal range validation failed |
| 5 | `FF_TIMEOUT` | Frame reception timeout |
| 6 | `FF_OVERFLOW` | Input buffer overflow detected |
| 7 | `FF_BIST_FAIL` | BIST loopback verification failed (optional, customer-dependent) |
| 8 | `FF_BIST_SPEAKER` | BIST speaker subsystem failure (optional, customer-dependent) |
| 9 | `FF_BIST_COUPLING` | BIST acoustic coupling degradation (optional, customer-dependent) |
| 10 | `FF_NOISE_PROFILE` | Noise profile anomaly detected (silence, DC rail, saturation, abnormal spectrum) |
| 11 | `FF_CROSSMIC` | Cross-microphone plausibility failure (channel divergence) |
| 12-15 | Reserved | Reserved for future use |

#### 4.2.5 Acoustic Loopback Output Interface (Optional, Customer-Dependent): AIQL → Speaker (via Codec DAC)

**Note**: This interface is only active when an exterior speaker (e.g., backup warning speaker, AVAS/pedestrian alerting speaker) is available on the vehicle platform. If no exterior speaker is present, this interface is inactive and all BIST-related functionality is disabled. See Section 7 (TSR-AIQL-015 through TSR-AIQL-018) for optional BIST TSRs.

| Field | Type | Size (bytes) | Description |
|-------|------|-------------|-------------|
| `bist_sequence_id` | uint16 | 2 | Unique BIST test sequence identifier |
| `signal_type` | enum8 | 1 | SINE_SWEEP (0x01), SINGLE_TONE (0x02) |
| `start_freq_hz` | uint16 | 2 | Sweep start frequency (default: 500 Hz) |
| `end_freq_hz` | uint16 | 2 | Sweep end frequency (default: 3000 Hz) |
| `amplitude_dbfs` | int8 | 1 | Output amplitude in dBFS (default: -20) |
| `duration_ms` | uint16 | 2 | Signal duration in milliseconds |
| `bist_samples` | int16[] | variable | PCM samples for speaker output |

#### 4.2.6 SysML Interface Block Definitions

The following interface blocks define the typed flows for all AIQL FlowPorts. Each maps to a SysML `«interfaceBlock»`.

```
«interfaceBlock» RawAudioFrame                «interfaceBlock» QualifiedAudioFrame
┌──────────────────────────────────┐          ┌─────────────────────────────────────┐
│ frame_id        : uint32         │          │ frame_id             : uint32       │
│ timestamp_us    : uint64         │          │ timestamp_us         : uint64       │
│ alive_counter   : uint8          │          │ qualification_status : enum8        │
│ sequence_counter: uint16         │          │   {QUALIFIED=0x01,                  │
│ sample_rate_hz  : uint32         │          │    DEGRADED=0x02,                   │
│ num_channels    : uint8          │          │    NOT_QUALIFIED=0x03}              │
│ num_samples     : uint16         │          │ audio_data           : int16[]      │
│ audio_data      : int16[]        │          │ classifier_output    : ClassifierOut│
│ classifier_output: ClassifierOut │          │ failure_flags        : uint16       │
│ crc32           : uint32         │          │ aiql_crc32           : uint32       │
└──────────────────────────────────┘          └─────────────────────────────────────┘

«interfaceBlock» ClassifierOutput             «interfaceBlock» BISTSignal
┌──────────────────────────────────┐          ┌─────────────────────────────────────┐
│ siren_probability : float32      │          │ (Optional, customer-dependent —     │
│ horn_probability  : float32      │          │  active only when exterior speaker  │
│ direction_deg     : float32      │          │  is available on the platform)      │
│ confidence        : float32      │          │                                     │
└──────────────────────────────────┘          │ bist_sequence_id : uint16           │
                                              │ signal_type      : enum8            │
«interfaceBlock» FailureFlags                 │   {SINE_SWEEP=0x01,                 │
┌──────────────────────────────────┐          │    SINGLE_TONE=0x02}                │
│ bit 0 : FF_FRESHNESS             │          │ start_freq_hz    : uint16 = 500     │
│ bit 1 : FF_ALIVE                 │          │ end_freq_hz      : uint16 = 3000    │
│ bit 2 : FF_CRC                   │          │ amplitude_dbfs   : int8   = -20     │
│ bit 3 : FF_SEQUENCE              │          │ duration_ms      : uint16           │
│ bit 4 : FF_RANGE                 │          │ bist_samples     : int16[]          │
│ bit 5 : FF_TIMEOUT               │          └─────────────────────────────────────┘
│ bit 6 : FF_OVERFLOW              │
│ bit 7 : FF_BIST_FAIL             │          «interfaceBlock» MRMRequest
│   (optional, customer-dependent) │          ┌─────────────────────────────────────┐
│ bit 8 : FF_BIST_SPEAKER          │          │ request_type  : enum8               │
│   (optional, customer-dependent) │          │   {BIST_STARTUP_FAIL,               │
│ bit 9 : FF_BIST_COUPLING         │          │    BIST_PERIODIC_FAIL,              │
│   (optional, customer-dependent) │          │    PERSISTENT_QUAL_FAIL}            │
│ bit 10 : FF_NOISE_PROFILE        │          │ failure_detail : uint16             │
│ bit 11 : FF_CROSSMIC             │          │ timestamp_us   : uint64             │
│ bit 12-15 : reserved             │          └─────────────────────────────────────┘
└──────────────────────────────────┘

«interfaceBlock» NoiseProfileResult
┌──────────────────────────────────┐
│ channel_id      : uint8          │
│ rms_level       : float32        │
│ dc_offset       : float32        │
│ clipping_pct    : float32        │
│ noise_floor_ok  : bool           │
│ spectral_ok     : bool           │
│ result          : enum8          │
│   {PASS, SILENCE, DC_RAIL,      │
│    SATURATION, ABNORMAL_SPEC}    │
└──────────────────────────────────┘

«interfaceBlock» DiagnosticEvent
┌──────────────────────────────────┐
│ event_type    : enum8            │
│   {STATE_TRANSITION,             │
│    FLAG_ASSERT,                  │
│    FLAG_DEASSERT,                │
│    BIST_RESULT,                  │
│    NOISE_PROFILE_ALERT,          │
│    CROSSMIC_ALERT,               │
│    RECOVERY}                     │
│ timestamp_us  : uint64           │
│ prev_state    : enum8            │
│ new_state     : enum8            │
│ failure_flags : uint16           │
│ bist_correlation : float32       │
└──────────────────────────────────┘
```

### 4.3 Timing Architecture

```
Audio Frame Period: 50 ms (20 Hz frame rate)

|<--- 50 ms frame period --->|
|                             |
[Audio Capture] [Processing] [AIQL Qualification] [AMO Ingestion]
|<-- 20 ms --->|<-- 10 ms ->|<----- 5 ms ------>|<----- 5 ms ------->|
                                                                       |
                              10 ms margin for jitter and scheduling --+

FTTI Budget Allocation (500 ms total):
  - Fault detection by AIQL:          <= 100 ms (2 audio frames)
  - State transition to DEGRADED:     <=  50 ms (1 frame)
  - AMO reaction (replan):  <= 150 ms
  - Vehicle dynamic response:         <= 200 ms
  Total:                                  500 ms

BIST Timing (Optional, Customer-Dependent — active only when exterior speaker available):
  Startup BIST:
    - Full sine sweep 500-3000 Hz:    2000 ms duration
    - Spectral match analysis:          50 ms
    - Total startup BIST:             2050 ms (must pass before QUALIFIED)
    - Retries on failure:             up to 3 (total max: ~8.2 s)

  Periodic BIST:
    - Burst signal (abbreviated sweep): 200 ms duration
    - Interval:                          every 60 seconds
    - Spectral match analysis:            50 ms
    - Signal cancellation for AMO: within same 200 ms window

  If no exterior speaker: startup qualification relies on passive checks
  (noise profile + cross-mic + sanitization) passing for configurable
  consecutive frames. No BIST timing applies.
```

### 4.4 SysML Block/Part Summary

The following table enumerates all SysML model elements for tool import (Cameo Systems Modeler, Rhapsody, Enterprise Architect, etc.).

| SysML Element | Stereotype | ASIL | Parent | Ports (FlowPort direction : type) | Status |
|---|---|---|---|---|---|
| `DRIVE_AGX_Thor_Platform` | `«block»` | — | *(top-level)* | — | Mandatory |
| `ExteriorMicArray` | `«block»` | QM | Customer | pOut_analogAudio : out AnalogAudio | Mandatory |
| `AudioCodec` | `«block»` | QM | Customer | pIn_analog : in AnalogAudio, pOut_a2b : out A2B_Frame, pIn_dacPcm : in BISTSignal | Mandatory |
| `A2B_Transceiver` | `«block»` | QM | Customer | pIn_a2b : in A2B_Frame, pOut_a2bFrame : out RawAudioFrame | Mandatory |
| `SirenHornClassifier` | `«block»` | QM | Customer | pIn_pcm : in AudioPCM, pOut_classResult : out ClassifierOutput | Mandatory |
| `ExteriorSpeaker` | `«block»` | QM | Customer | pIn_dacSignal : in AnalogAudio | Optional |
| **`AIQL`** | **`«block»`** | **B** | Thor Main (BP) | pIn_rawFrame : in RawAudioFrame, pOut_qualFrame : out QualifiedAudioFrame, pOut_safetyStatus : out SafetyStatus, pOut_bistSignal : out BISTSignal (optional), pOut_diagLog : out DiagnosticEvent | Mandatory |
| `AIQL::inputDemux` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::freshnessCheck` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::aliveCheck` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::crcCheck` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::seqCheck` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::rangeValidator` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::passiveMonitor` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::passiveMonitor::noiseProfileCheck` | `«part»` | B | passiveMonitor | — | Mandatory |
| `AIQL::passiveMonitor::crossMicPlausibility` | `«part»` | B | passiveMonitor | — | Mandatory |
| `AIQL::passiveMonitor::inputSanitizer` | `«part»` | B | passiveMonitor | — | Mandatory |
| `AIQL::qualificationDecision` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::stateMachine` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::outputAssembler` | `«part»` | B | AIQL | — | Mandatory |
| `AIQL::bistModule` | `«part»` | B | AIQL | — | Optional |
| `AIQL::bistModule::signalGenerator` | `«part»` | B | bistModule | — | Optional |
| `AIQL::bistModule::spectralMatchVerifier` | `«part»` | B | bistModule | — | Optional |
| `AIQL::bistModule::signalCanceller` | `«part»` | B | bistModule | — | Optional |
| `AIQL::bistModule::bistRetryController` | `«part»` | B | bistModule | — | Optional |
| `AIQL::hwProtection` | `«part»` | D | AIQL | *(provided by DRIVE OS)* | Mandatory |
| `AMO` | `«block»` | Mixed | Thor Main | pIn_qualFrame : in QualifiedAudioFrame | Mandatory |
| `THOR_FSI` | `«block»` | D | BP | pIn_safetyStatus : in SafetyStatus (via SOC_Error / FSI_I2C) | Mandatory |
| `DRIVE_OS` | `«block»` | D | Platform | — | Mandatory |

### 4.5 Data Flow Sequences

The following sequences define the nominal and BIST processing flows. Map each to a SysML **Sequence Diagram** or **Activity Diagram**.

#### 4.5.1 Nominal Frame Processing (every 50 ms)

```
A2B Transceiver          AIQL (Thor Main, BP SW)                       AMO        THOR FSI
    |                      |                                              |            |
    |  rawAudioFrame       |                                              |            |
    |  (A2B, 20 Hz)        |                                              |            |
    |─────────────────────>|                                              |            |
    |                      |                                              |            |
    |                      |──> inputDemux                                |            |
    |                      |     |                                        |            |
    |                      |     |── timestamp_us ──> freshnessCheck      |            |
    |                      |     |── alive_ctr    ──> aliveCheck          |            |
    |                      |     |── all_fields   ──> crcCheck            |            |
    |                      |     |── seq_ctr      ──> seqCheck            |            |
    |                      |     |── audio_data   ──> rangeValidator      |            |
    |                      |     |   + classifier ──> rangeValidator      |            |
    |                      |     |── audio_data   ──> passiveMonitor      |            |
    |                      |     |   (noise profile + cross-mic + sanitize)|            |
    |                      |                                              |            |
    |                      |  checkResults[5] + passiveResults            |            |
    |                      |     |                                        |            |
    |                      |     v                                        |            |
    |                      |  qualificationDecision                       |            |
    |                      |     | + passiveResult (from passiveMonitor)  |            |
    |                      |     | + bistResult (optional, from bistModule)|            |
    |                      |     | + currentState (from stateMachine)     |            |
    |                      |     |                                        |            |
    |                      |     |--> stateTransitionCmd                  |            |
    |                      |     |       |                                |            |
    |                      |     |       v                                |            |
    |                      |     |    stateMachine.evaluate()             |            |
    |                      |     |       |                                |            |
    |                      |     |       |--> qualification_status        |            |
    |                      |     |       |--> safetyStatusUpdate (if any) |            |
    |                      |     |                                        |            |
    |                      |     |--> aggregateFlags                      |            |
    |                      |             |                                |            |
    |                      |             v                                |            |
    |                      |          outputAssembler.build()             |            |
    |                      |             |                  |             |            |
    |                      |             | QualifiedFrame   | SafetyStatus|            |
    |                      |  pOut_qualFrame                |             |            |
    |                      |─────────────────────────────-->|             |            |
    |                      |                      pOut_safetyStatus       |            |
    |                      |─────────────────────────────────────────────>|            |
    |                      |                          (SOC_Error/FSI_I2C) |            |
```

#### 4.5.2 Startup BIST Sequence (Optional, Customer-Dependent — at power-on, before QUALIFIED)

**Note**: This sequence only applies when an exterior speaker is available on the vehicle platform. If no speaker is present (Hyperion reference platform), startup qualification relies on passive checks (TSR-AIQL-020, -021, -022) passing for a configurable number of consecutive frames.

```
AIQL                  Codec (DAC)    Exterior      Codec (ADC)     AIQL
bistModule            QM HW          Speaker       + Mic Array     bistModule
    |                    |           QM HW             |               |
    |                    |              |               |               |
    | signalGenerator    |              |               |               |
    | generates 2000 ms  |              |               |               |
    | sine sweep         |              |               |               |
    | 500-3000 Hz        |              |               |               |
    | -20 dBFS           |              |               |               |
    |                    |              |               |               |
    |  BISTSignal        |              |               |               |
    |  (pOut_bistSignal) |              |               |               |
    |───────────────────>|              |               |               |
    |                    | DAC output   |               |               |
    |                    |─────────────>|               |               |
    |                    |              | acoustic      |               |
    |                    |              | propagation   |               |
    |                    |              | (exterior     |               |
    |                    |              |  air path)    |               |
    |                    |              |──────────────>|               |
    |                    |              |               | ADC capture   |
    |                    |              |               | (via normal   |
    |                    |              |               |  A2B path)    |
    |                    |              |               |               |
    |                    |              |         rawAudioFrame with    |
    |                    |              |         BIST signal captured  |
    |<─────────────────────────────────────────────────────────────────|
    |                                                                  |
    | spectralMatchVerifier:                                           |
    |  1. FFT(captured_mic_signal)                                     |
    |  2. FFT(known_reference_signal)                                  |
    |  3. normalized_cross_correlation                                 |
    |  4. 1/3-octave band amplitude check                              |
    |                                                                  |
    |  correlation >= 0.85?  ──YES──> PASS ──> stateMachine: QUALIFIED |
    |                        ──NO───> FAIL                             |
    |                                   |                              |
    |  bistRetryController:             |                              |
    |    retry count < 3? ────YES────> retry (loop back to sweep)      |
    |                     ────NO─────> FAIL CONFIRMED                  |
    |                                   |                              |
    |                                   v                              |
    |                          stateMachine: MRM_REQUESTED             |
    |                          pOut_safetyStatus ──> THOR FSI          |
    |                                                                  |
```

#### 4.5.3 Periodic BIST Sequence (Optional, Customer-Dependent — every 60 s during QUALIFIED)

**Note**: This sequence only applies when an exterior speaker is available. If no speaker, continuous passive monitoring (TSR-AIQL-020, -021, -022) provides ongoing diagnostic coverage.

```
AIQL              Codec (DAC)    Exterior    Mic Array      AIQL           AMO
bistModule        QM HW          Speaker     + Codec        bistModule     (Alpamayo)
    |                |           QM HW          |               |             |
    | [timer: 60 s]  |              |            |               |             |
    |                |              |            |               |             |
    | signalGenerator|              |            |               |             |
    | 200 ms burst   |              |            |               |             |
    | (abbreviated)  |              |            |               |             |
    |                |              |            |               |             |
    |  BISTSignal    |              |            |               |             |
    |───────────────>|──────────────>|───acoustic─>|              |             |
    |                |              |            |               |             |
    |                |              |      rawAudioFrame         |             |
    |                |              |      (BIST + real audio)   |             |
    |<──────────────────────────────────────────────────────────|             |
    |                                                           |             |
    | signalCanceller:                                          |             |
    |   cleaned_audio = raw_audio - known_bist_signal           |             |
    |   attenuation >= 30 dB                                    |             |
    |                                                           |             |
    |   cleaned_audio ──> outputAssembler ──> pOut_qualFrame ──────────────-->|
    |                                                           |             |
    | spectralMatchVerifier (simultaneous):                     |             |
    |   correlation >= 0.85?                                    |             |
    |                                                           |             |
    |   YES ──> PASS ──> continue QUALIFIED, reset retry count  |             |
    |   NO  ──> FAIL ──> bistRetryController                    |             |
    |                      retry count < 2? ──YES──> retry      |             |
    |                                       ──NO──> FAIL CONFIRMED            |
    |                                                  |                      |
    |                                      stateMachine: MRM_REQUESTED        |
    |                                      pOut_safetyStatus ──> THOR FSI     |
    |                                                                         |
```

---

## 5. Safety Goal & ASIL Determination

### 5.1 Safety Goal Definition

**SG-EMV-01: The autonomous vehicle shall avoid collision with activated emergency vehicles (siren + flashing lights) driving slowly or stationary in front on access-controlled highways.**

| Attribute | Value |
|-----------|-------|
| Safety Goal ID | SG-EMV-01 |
| Item | Audio Input Path (Microphone → Codec → Driver → AIQL → AMO) |
| Hazardous Event | Collision with emergency vehicle in operation (siren + flashing lights) that is slow or stationary in front of the ego vehicle on an access-controlled highway with middle separation |
| Collision Type | Side-front collision at highway speed |
| Sensor Role | **Secondary** — audio (siren detection) supplements primary sensors (camera for flashing lights/vehicle shape, LiDAR/radar for stationary/slow object) for emergency vehicle detection. AMO multi-sensor fusion uses audio as an additional input channel; driving decisions are not solely dependent on audio. |
| HARA Reference | Audio_HARA / UC-React-to-EMV — Row: "Yield to Activated EMV (Highway), siren+lights, slow/stationary in front" |

#### HARA Context — Related Scenarios

The HARA (Audio_HARA.xlsx, sheet UC-React-to-EMV) analyzes multiple emergency vehicle interaction scenarios. The following table summarizes the complete HARA landscape and identifies which safety goal the AIQL supports:

| Scenario | S | E | C | ASIL | Safety Goal | Audio Relevant? |
|---|---|---|---|---|---|---|
| Highway: EMV siren+lights **from behind** | S0 | — | C0 | Not safety critical | No SG | No — trailing vehicle has controllability |
| Highway: EMV lights only **from behind** | S0 | — | C0 | Not safety critical | No SG | No |
| Highway: EMV wrong direction (ghostdriver) | S3 | E1 | C2 | QM | Avoid collision w/ EMV in wrong dir. | No — QM, no safety goal |
| **Highway: EMV siren+lights, slow/stationary in front** | **S3** | **E2** | **C3** | **ASIL B** | **SG-EMV-01 (this TSC)** | **Yes — siren is supplementary detection** |
| Highway: Warning-signal vehicles in front (no siren) | S3 | E3 | C3 | ASIL C | Avoid collision w/ warning-signal veh. | No — lights only, no siren |
| Urban intersection, low speed (<50 kph) | S2 | E2 | C1 | QM | — | No — QM |
| Urban intersection, medium (50-70 kph) | S3 | E2 | C1 | QM | — | No — QM |
| Urban intersection, high (70-90 kph) | S3 | E2 | C1→C2 | QM→ASIL A | Avoid collision w/ crossing EMV | Marginal — ASIL A, lower priority |

The AIQL addresses **SG-EMV-01** (ASIL B). The ASIL C scenario (warning-signal vehicles) does not involve sirens and is outside the AIQL scope — it is addressed by visual detection at the system level.

### 5.2 ASIL Determination

The ASIL for SG-EMV-01 is determined per ISO 26262-3 Table 4 using the parameters from the HARA (Audio_HARA.xlsx):

#### Severity: S3 (Life-threatening injuries, survival uncertain)

**HARA justification**: Side-front collision at highway speed between the ego vehicle and a stationary or slow-moving emergency vehicle on an access-controlled highway. At highway speeds (≥ 100 km/h ego), the collision energy is sufficient to cause life-threatening injuries. The emergency vehicle may be stationary (blocking a lane after responding to an incident) or moving slowly, resulting in high closing speed and severe collision dynamics.

#### Exposure: E2 (Low probability)

**HARA justification (VDA 702 frequency exposure: 1 ≤ x < 10/year)**:
- Activated emergency vehicles (siren + flashing lights) stationary or slow in front on access-controlled highways are encountered infrequently
- Not every highway trip involves such an encounter
- The specific scenario requires the EMV to be in the ego vehicle's lane or obstructing the travel path on a highway with middle separation
- E3 is not assigned because the combined conditions (highway + EMV in front + same travel direction + siren active) have lower frequency than general EMV encounters

#### Controllability: C3 (Difficult to control or uncontrollable)

**HARA justification**: "No driver" — this is a Level 4 autonomous vehicle with no human driver available to take control. The vehicle must handle the situation autonomously. C3 is the correct classification for all Level 4/5 AV scenarios where no human fallback is available.

#### ASIL Result

Per ISO 26262-3 Table 4: **S3 × E2 × C3 = ASIL B**

This is confirmed by the HARA (Audio_HARA.xlsx, column K: "B").

### 5.3 Fault Tolerant Time Interval (FTTI)

**FTTI: 500 ms**

**Justification**: The HARA scenario involves an EMV that is slow or stationary in front on a highway. While the closing speed is high, the EMV is a large, visually conspicuous object (flashing lights, distinctive shape) detectable at significant range by primary sensors. The 500 ms FTTI for the audio subsystem is justified because:

1. **Primary sensor coverage**: Camera, LiDAR, and radar detect the stationary/slow EMV independently of audio. Audio failure does not create a detection gap — it reduces fusion confidence for EMV classification
2. **Multiple detection opportunities**: At 20 Hz audio frame rate, 500 ms provides 10 audio frames for fault detection
3. **Secondary sensor budget**: Audio contributes to earlier EMV classification (siren confirms the visual object is an EMV in operation), but the collision avoidance decision is driven by primary sensor object detection at much longer range than audio detection range
4. **HARA scenario dynamics**: The ego vehicle approaches the stationary/slow EMV on the highway over several seconds; the NDAS DRIVE AV Safety Force Field will independently trigger braking for any stationary obstacle regardless of EMV classification
5. **Vehicle dynamics**: Avoidance involves lateral displacement (lane change) or controlled deceleration, both of which have response times on the order of seconds at highway distances

The 500 ms FTTI must be confirmed by system-level timing analysis (see Open Items, Section 14).

---

## 6. Failure Modes & Freedom From Interference

### 6.1 Audio Input Failure Mode Catalog

The following failure modes are identified for the QM audio input path. Each failure mode is analyzed for its effect on AMO emergency vehicle detection capability.

#### FM-01: Complete Loss of Audio Data

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | No audio frames received by AIQL |
| **Failure Mechanism** | Microphone hardware failure; codec power loss; driver crash; bus failure; memory allocation failure |
| **Effect on AMO** | Total loss of siren/horn detection; emergency vehicle detection relies solely on camera |
| **Detection Mechanism** | Frame reception timeout (TSR-AIQL-001), alive counter missing (TSR-AIQL-002); noise profile monitoring — silence detection (TSR-AIQL-020); cross-mic plausibility — all channels simultaneously lost (TSR-AIQL-021); BIST loopback detection (TSR-AIQL-016, optional/customer-dependent); graceful degradation (TSR-AIQL-007) |
| **Safety Impact** | Potential violation of SG-EMV-01 if camera-only detection is insufficient |
| **Frequency Estimate** | Low (hardware failure rate ~500 FIT for complete path loss) |

#### FM-02: Audio Data Corruption

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio frame data is corrupted during transmission or processing |
| **Failure Mechanism** | Memory bit-flip (SEU); DMA error; buffer overrun in driver; EMI-induced data corruption on I2S bus |
| **Effect on AMO** | Corrupted audio may cause false siren detections (false positive) or mask real sirens (false negative) |
| **Detection Mechanism** | CRC-32 integrity check (TSR-AIQL-003), range validation (TSR-AIQL-005); graceful degradation (TSR-AIQL-007) |
| **Safety Impact** | False positive: unwarranted yielding (nuisance). False negative: violates SG-EMV-01 |
| **Frequency Estimate** | Medium (SEU rate ~200 FIT; EMI susceptibility dependent on shielding) |

#### FM-03: Audio Data Staleness

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio frames are received but contain outdated data (repeated or delayed frames) |
| **Failure Mechanism** | Driver buffer stall; scheduling priority inversion; DMA descriptor not updated; timestamp rollover |
| **Effect on AMO** | Stale audio data may show siren present when it has passed, or not-present when it has arrived |
| **Detection Mechanism** | Freshness check (TSR-AIQL-001), alive counter validation (TSR-AIQL-002), sequence counter monotonicity (TSR-AIQL-004); graceful degradation (TSR-AIQL-007) |
| **Safety Impact** | Delayed yielding response — violation of SG-EMV-01 FTTI |
| **Frequency Estimate** | Medium (scheduling issues ~100 FIT; increases under system load) |

#### FM-04: Audio Signal Out-of-Range

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio signal values exceed physical plausibility bounds |
| **Failure Mechanism** | Microphone bias failure; codec gain register corruption; ADC saturation; reference voltage drift |
| **Effect on AMO** | Clipped or offset audio distorts frequency content, leading to classifier errors |
| **Detection Mechanism** | Range validation (TSR-AIQL-005) — sample amplitude, DC offset, noise floor checks; noise profile monitoring — DC rail, persistent saturation, abnormal spectral content (TSR-AIQL-020); cross-mic plausibility — single channel deviating from peers indicates hardware fault (TSR-AIQL-021); BIST spectral match detects analog path degradation (TSR-AIQL-016, optional/customer-dependent) |
| **Safety Impact** | Classifier output unreliable — potential SG-EMV-01 violation |
| **Frequency Estimate** | Low (analog fault rate ~50 FIT; codec register corruption ~20 FIT) |

#### FM-05: False Negative (Missed Siren Detection)

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Real siren present but classifier reports no siren |
| **Failure Mechanism** | QM classifier systematic error; ambient noise masking; Doppler shift moving siren out of classifier frequency band; microphone sensitivity degradation |
| **Effect on AMO** | Reduced supplementary input for emergency vehicle detection — AMO continues to detect via primary sensors (camera, LiDAR, radar) |
| **Detection Mechanism** | **Not addressed by AIQL** — this is a classifier performance limitation, not a data integrity failure. As audio is a secondary sensor, FM-05 is mitigated at the system level by AMO multi-sensor fusion with primary sensors. The AIQL qualifies data integrity but does not validate classifier semantic correctness. |
| **Safety Impact** | Low at AIQL level — primary sensors provide independent emergency vehicle detection. Residual risk: reduced detection confidence in scenarios where audio would have provided early warning (e.g., emergency vehicle approaching from behind a visual obstruction). |
| **Frequency Estimate** | Medium-High (systematic failure; not quantifiable by FIT rate alone) |

#### FM-06: False Positive (Phantom Siren Detection)

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | No siren present but classifier reports siren detected |
| **Failure Mechanism** | QM classifier systematic error; environmental sounds resembling sirens (construction equipment, musical instruments, other vehicle horns); acoustic reflections causing phantom source |
| **Effect on AMO** | AMO receives false siren indication — effect is limited because audio is a secondary sensor and AMO's fusion weighs primary sensor inputs (camera, LiDAR, radar) for driving decisions |
| **Detection Mechanism** | **Not addressed by AIQL** — this is a classifier performance limitation, not a data integrity failure. As audio is a secondary sensor, FM-06 is mitigated at the system level by AMO multi-sensor fusion. Primary sensors provide independent confirmation; a false audio-only siren detection without corroborating primary sensor data will be downweighted by fusion. |
| **Safety Impact** | Low — unwarranted yielding requires corroboration from primary sensors in AMO's fusion architecture. Audio-only siren detection without primary sensor confirmation does not trigger yielding. |
| **Frequency Estimate** | Medium (systematic failure; dependent on acoustic environment) |

#### FM-07: Sequence Disorder

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio frames arrive out of sequence or with duplicated sequence numbers |
| **Failure Mechanism** | Multi-threaded driver race condition; DMA descriptor ring corruption; buffer management defect; interrupt priority inversion |
| **Effect on AMO** | Out-of-order frames disrupt temporal tracking of siren presence/absence transitions |
| **Detection Mechanism** | Sequence counter monotonicity check (TSR-AIQL-004) |
| **Safety Impact** | Delayed or incorrect siren detection state — potential SG-EMV-01 violation |
| **Frequency Estimate** | Low (software defect; ~10 FIT under normal scheduling) |

#### FM-08: Common Cause Failure

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Multiple audio channels or both audio and AIQL fail simultaneously |
| **Failure Mechanism** | Shared power rail failure; shared clock source failure; SoC-wide thermal shutdown; common mode EMI affecting all audio channels; shared memory controller failure |
| **Effect on AMO** | Complete loss of qualified audio with potential AIQL integrity compromise |
| **Detection Mechanism** | Graceful degradation (TSR-AIQL-007), spatial FFI MPU (TSR-AIQL-012), temporal FFI watchdog (TSR-AIQL-013); diagnostic logging (TSR-AIQL-011) |
| **Safety Impact** | **Critical** — defeats both the QM element and the safety mechanism simultaneously |
| **Frequency Estimate** | Very Low (common cause beta factor ~2% of single-point failure rate) |

#### FM-09: Speaker Subsystem Failure (Optional, Customer-Dependent)

**Note**: This failure mode only applies when an exterior speaker is available on the vehicle platform for acoustic loopback BIST. If no speaker is present (Hyperion reference platform does not include an exterior speaker for BIST purposes), this failure mode does not apply and passive qualification checks (noise profile, cross-mic plausibility, input sanitization) serve as the primary diagnostic approach.

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Exterior speaker cannot produce BIST test signal |
| **Failure Mechanism** | Speaker driver IC failure; speaker coil open/short; codec DAC failure; wiring harness disconnection; amplifier fault |
| **Effect on AMO** | BIST cannot execute — loss of acoustic loopback coverage for FM-01 and FM-04. Passive qualification checks (TSR-AIQL-020, TSR-AIQL-021, TSR-AIQL-022) continue to provide diagnostic coverage. |
| **Detection Mechanism** | BIST output monitoring (TSR-AIQL-015) — expected signal not detected at microphone within timeout; FF_BIST_SPEAKER flag set |
| **Safety Impact** | Loss of optional BIST diagnostic coverage — degrades confidence in end-to-end acoustic path integrity. Triggers MRM after retry exhaustion (TSR-AIQL-019) if BIST is enabled. |
| **Frequency Estimate** | Low (speaker failure rate ~100 FIT; amplifier failure ~50 FIT) |

#### FM-10: Acoustic Coupling Degradation (Optional, Customer-Dependent)

**Note**: This failure mode only applies when acoustic loopback BIST is enabled (exterior speaker available). If no speaker is present, acoustic coupling between speaker and microphone is not relevant. Microphone-side degradation (sensitivity drift, membrane contamination) is instead detected by passive checks: noise profile monitoring (TSR-AIQL-020) and cross-mic plausibility (TSR-AIQL-021).

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | BIST spectral match fails due to degraded acoustic coupling between exterior speaker and microphones |
| **Failure Mechanism** | Physical obstruction (debris over speaker/microphone); microphone membrane contamination; speaker baffle deformation; environmental exposure affecting acoustic path; microphone sensitivity drift |
| **Effect on AMO** | BIST indicates audio path degradation — correlation below threshold suggests microphone or acoustic path has changed, potentially affecting siren detection quality |
| **Detection Mechanism** | BIST spectral match correlation check (TSR-AIQL-016) — correlation < 0.85 or amplitude deviation > +/- 6 dB; FF_BIST_COUPLING flag set |
| **Safety Impact** | Indicates potential degradation of siren detection capability. Triggers MRM after retry exhaustion (TSR-AIQL-019) if BIST is enabled. |
| **Frequency Estimate** | Medium (environmental contamination ~200 FIT; sensitivity drift ~50 FIT over lifetime) |

#### FM-11: BIST False Failure (Optional, Customer-Dependent)

**Note**: This failure mode only applies when acoustic loopback BIST is enabled (exterior speaker available). If no speaker is present, BIST is not executed and this failure mode does not apply. For platforms without BIST, the false failure risk shifts to passive checks (noise profile and cross-mic plausibility), which have lower false-positive rates because they do not inject a test signal into the acoustic environment.

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | BIST spectral match fails despite healthy audio path due to environmental noise interference |
| **Failure Mechanism** | Wind noise during BIST window; traffic noise masking BIST signal; road surface noise at BIST frequencies; environmental noise during periodic BIST |
| **Effect on AMO** | False BIST failure triggers unnecessary MRM — availability impact (vehicle stops when audio path is actually functional) |
| **Detection Mechanism** | Retry mechanism (TSR-AIQL-017, TSR-AIQL-019) — 3 retries at startup, 2 retries during periodic BIST; signal-to-interference analysis during spectral match |
| **Safety Impact** | Availability impact only — false MRM is a nuisance but not a safety hazard. Retry mechanism reduces false failure probability. |
| **Frequency Estimate** | Medium (dependent on environmental noise; mitigated by BIST signal level at -20 dBFS and retry logic) |

#### FM-12: Microphone Blockage/Environmental Damage

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | One or more exterior microphone channels are blocked or damaged by environmental exposure |
| **Failure Mechanism** | Ice accumulation on microphone port; mud/debris covering microphone; water ingress into microphone cavity; road spray coating; insect or bird nesting material blockage; UV degradation of microphone membrane |
| **Effect on AMO** | Blocked or damaged microphone produces attenuated, distorted, or absent audio on affected channel(s); siren detection capability degraded on affected channels; if all channels affected, total loss of siren detection |
| **Detection Mechanism** | Noise floor analysis (TSR-AIQL-020) — blocked mic exhibits abnormally low noise floor (silence) or altered spectral characteristics; cross-mic plausibility (TSR-AIQL-021) — blocked channel deviates significantly from peers in RMS level and spectral shape; input sanitization (TSR-AIQL-022) — frames from blocked mic may exhibit persistent low-amplitude or DC-biased signals |
| **Safety Impact** | Partial degradation if subset of mics affected (remaining mics maintain siren detection). Complete loss if all mics blocked — detected and mitigated by transition to NOT_QUALIFIED. |
| **Frequency Estimate** | Medium (exterior mounting increases exposure to environmental hazards; frequency depends on operating environment and vehicle speed) |

#### FM-13: Cross-Mic Plausibility Failure

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | One or more microphone channels deviate significantly from peer channels in RMS level or spectral characteristics |
| **Failure Mechanism** | Individual microphone hardware degradation (sensitivity drift, preamp failure); individual channel wiring fault (intermittent connection, increased resistance); individual codec ADC channel failure; per-channel gain register corruption |
| **Effect on AMO** | Divergent channel provides inconsistent audio data that may confuse direction-of-arrival estimation or reduce siren detection confidence; if majority of channels diverge, overall audio quality compromised |
| **Detection Mechanism** | Cross-mic plausibility check (TSR-AIQL-021) — compare RMS levels and spectral characteristics across all mic channels; flag any channel deviating by more than configurable threshold (default: X dB) from the median of its peers. Requires minimum 2 microphones for comparison. |
| **Safety Impact** | Individual channel divergence: DEGRADED state, AMO continues with remaining qualified channels. Multiple channel divergence: NOT_QUALIFIED state, audio excluded from fusion. |
| **Frequency Estimate** | Low (individual channel hardware failure ~50 FIT per channel; increases with number of channels and exposure to vibration/thermal cycling) |

### 6.2 Freedom From Interference (FFI) Analysis

FFI analysis ensures that failures in the QM audio subsystem cannot propagate into and corrupt the ASIL B AIQL or the ASIL-rated AMO system. Three FFI dimensions are analyzed per ISO 26262-9.

#### 6.2.1 Spatial FFI (Freedom from Interference in Memory/Data)

| Concern | Mechanism | Requirement |
|---------|-----------|-------------|
| QM audio driver writes to AIQL memory space | MPU/MMU hardware isolation; AIQL memory region marked read-only to QM processes | TSR-AIQL-012 |
| Shared memory buffer corruption by QM DMA | DMA firewall restricts QM audio DMA to designated audio buffer region; AIQL reads from this region but QM cannot write to AIQL code/data | TSR-AIQL-012 |
| Stack/heap overflow in QM driver corrupting AIQL data | Separate memory partitions; guard pages between QM and ASIL partitions | TSR-AIQL-012 |

#### 6.2.2 Temporal FFI (Freedom from Interference in Execution Time)

| Concern | Mechanism | Requirement |
|---------|-----------|-------------|
| QM audio driver consumes excessive CPU time, starving AIQL | AIQL runs on a dedicated CPU core or in a time-partitioned scheduler with guaranteed execution budget | TSR-AIQL-013 |
| QM driver enters infinite loop, blocking shared resource | Hardware watchdog monitors AIQL execution; WCET analysis ensures AIQL completes within budget | TSR-AIQL-009, TSR-AIQL-013 |
| Interrupt storm from audio hardware prevents AIQL scheduling | Interrupt rate limiting on audio IRQ; AIQL uses polling or dedicated interrupt with higher priority | TSR-AIQL-013 |

#### 6.2.3 Communication FFI (Freedom from Interference in Data Exchange)

| Concern | Mechanism | Requirement |
|---------|-----------|-------------|
| Corrupted data passed from QM to AIQL via shared memory | CRC-32 end-to-end protection; AIQL independently verifies CRC before processing | TSR-AIQL-003 |
| Stale data in shared memory buffer | Freshness timestamp and alive counter; AIQL rejects frames older than 100 ms | TSR-AIQL-001, TSR-AIQL-002 |
| Replayed or reordered frames | Sequence counter monotonicity check; AIQL rejects frames with non-monotonic sequence numbers | TSR-AIQL-004 |
| BIST signal interference with qualified audio data | BIST signal cancellation during periodic BIST; startup BIST executes before QUALIFIED state | TSR-AIQL-018 |

---

## 7. Technical Safety Requirements

### TSR-AIQL-001: Freshness Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-001 |
| **Requirement** | The AIQL shall reject any audio frame whose timestamp is older than 100 ms relative to the current system time. |
| **ASIL** | B |
| **Rationale** | At 20 Hz frame rate (50 ms period), a 100 ms threshold allows for up to one missed frame before declaring staleness. A stale frame represents data that may no longer reflect the current acoustic environment, potentially causing delayed siren detection. The timeout also detects complete loss of audio frames (no new frames received). |
| **Failure Mode Addressed** | FM-01 (Loss), FM-03 (Staleness) |
| **Verification** | FI-01, FI-02 |
| **Acceptance Criterion** | 100% of frames with timestamp > 100 ms stale are rejected; zero false rejections of fresh frames under nominal jitter (< 10 ms). |

### TSR-AIQL-002: Alive Counter Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-002 |
| **Requirement** | The AIQL shall verify that the alive counter in each received audio frame increments by exactly 1 (modulo 256) compared to the previously accepted frame. The AIQL shall transition to DEGRADED state if 3 consecutive alive counter violations are detected. |
| **ASIL** | B |
| **Rationale** | The alive counter provides a lightweight liveness indication from the QM audio driver. Three consecutive violations (150 ms at 20 Hz) provides robustness against transient scheduling jitter while detecting sustained driver failures within the FTTI. |
| **Failure Mode Addressed** | FM-01 (Loss), FM-03 (Staleness) |
| **Verification** | FI-01 |
| **Acceptance Criterion** | Alive counter violations detected within 1 frame; DEGRADED state entered within 150 ms of sustained violation. |

### TSR-AIQL-003: CRC-32 Integrity Check

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-003 |
| **Requirement** | The AIQL shall compute a CRC-32 over all received audio frame fields (excluding the CRC field itself) and compare it to the transmitted CRC-32. Frames with CRC mismatch shall be discarded and the failure flag FF_CRC shall be set. |
| **ASIL** | B |
| **Rationale** | CRC-32 provides a Hamming distance of 4 for messages up to 3.4 GB, sufficient to detect all 1-bit, 2-bit, and 3-bit errors, and most burst errors up to 32 bits. This addresses data corruption from SEU, DMA errors, and EMI-induced bit flips. Per AUTOSAR E2E Profile 1. |
| **Failure Mode Addressed** | FM-02 (Corruption) |
| **Verification** | FI-03 |
| **Acceptance Criterion** | 100% detection of single-bit errors; > 99.99% detection of random multi-bit errors; zero false CRC failures on uncorrupted frames. |

### TSR-AIQL-004: Sequence Counter Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-004 |
| **Requirement** | The AIQL shall verify that the sequence counter of each received audio frame is strictly greater than the sequence counter of the previously accepted frame (accounting for 16-bit wraparound). Frames with non-monotonic sequence counters shall be discarded and the failure flag FF_SEQUENCE shall be set. |
| **ASIL** | B |
| **Rationale** | Sequence counter monotonicity detects replayed frames, reordered frames, and duplicate frames. The 16-bit counter wraps after 65535 frames (~55 minutes at 20 Hz), which is acceptable for detecting transient sequence errors. |
| **Failure Mode Addressed** | FM-07 (Sequence Disorder) |
| **Verification** | FI-04 |
| **Acceptance Criterion** | 100% detection of replayed, reordered, and duplicated frames. |

### TSR-AIQL-005: Range Validation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-005 |
| **Requirement** | The AIQL shall validate the following signal ranges for each audio frame and set FF_RANGE if any check fails: (a) Audio sample amplitudes within [-32768, +32767] (int16 range) with no more than 5% of samples at saturation (|value| > 32000); (b) DC offset of frame mean within [-500, +500] LSB; (c) Noise floor RMS above 10 LSB (microphone connectivity check); (d) Classifier probabilities within [0.0, 1.0]; (e) Direction-of-arrival within [-180.0, +180.0] degrees; (f) Classifier confidence within [0.0, 1.0]. |
| **ASIL** | B |
| **Rationale** | Range checks detect analog hardware faults (bias failure, gain corruption, ADC saturation, microphone disconnection) and data format errors (NaN, infinity, out-of-bounds values). The 5% saturation threshold allows for genuine loud events while detecting persistent clipping from hardware faults. The noise floor check detects disconnected or shorted microphones. |
| **Failure Mode Addressed** | FM-04 (Out-of-Range) |
| **Verification** | FI-05 |
| **Acceptance Criterion** | Detection of all out-of-range conditions within 1 frame; zero false range violations for signals within specified bounds. |

### TSR-AIQL-007: Graceful Degradation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-007 |
| **Requirement** | When the AIQL qualification state transitions to NOT_QUALIFIED, the AIQL shall: (a) Output qualification_status = NOT_QUALIFIED to AMO; (b) Clear the audio_data and classifier_output fields (zero-fill); (c) Continue outputting frames at 20 Hz to maintain alive signaling; (d) Set all applicable failure_flags bits. AMO shall exclude audio from sensor fusion and continue emergency vehicle detection using primary sensors (camera, LiDAR, radar) upon receiving NOT_QUALIFIED status. When NOT_QUALIFIED is caused by confirmed BIST failure (FF_BIST_FAIL, FF_BIST_SPEAKER, or FF_BIST_COUPLING set after retry exhaustion), the AIQL shall additionally request MRM transition per TSR-AIQL-019. |
| **ASIL** | B |
| **Rationale** | Graceful degradation ensures that audio failures result in a deterministic exclusion of audio from AMO sensor fusion rather than uncontrolled behavior. Zero-filling audio data prevents AMO from using corrupted or stale data. Continued 20 Hz output ensures AMO can distinguish "audio not qualified" from "AIQL has crashed." As audio is a secondary sensor, the transition to primary-sensor-only operation is a minimal degradation. |
| **Failure Mode Addressed** | FM-01 (Loss), FM-02 (Corruption), FM-03 (Staleness), FM-08 (Common Cause), FM-09 (Speaker Failure), FM-10 (Coupling Degradation) |
| **Verification** | FI-07 |
| **Acceptance Criterion** | NOT_QUALIFIED status transmitted within 100 ms of state transition; zero non-zero audio data bytes in NOT_QUALIFIED frames. |

### TSR-AIQL-008: Recovery Criteria

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-008 |
| **Requirement** | The AIQL shall transition from DEGRADED to QUALIFIED only after receiving 10 consecutive valid audio frames (500 ms at 20 Hz) with all checks passing (freshness, alive, CRC, sequence, range). The AIQL shall transition from NOT_QUALIFIED to DEGRADED only after receiving 20 consecutive valid audio frames (1000 ms) with all checks passing. |
| **ASIL** | B |
| **Rationale** | Hysteresis in recovery prevents oscillation between qualification states during intermittent fault conditions. The asymmetric recovery times (500 ms from DEGRADED, 1000 ms from NOT_QUALIFIED) reflect increasing caution for more severe failure states. Recovery from NOT_QUALIFIED requires a longer observation window to confirm stable audio path operation. |
| **Failure Mode Addressed** | All failure modes (recovery behavior) |
| **Verification** | FI-07 |
| **Acceptance Criterion** | No recovery from DEGRADED in fewer than 10 consecutive valid frames. No recovery from NOT_QUALIFIED in fewer than 20 consecutive valid frames. No oscillation (> 2 state transitions within 5 seconds) during steady-state fault injection. |

### TSR-AIQL-009: Worst-Case Execution Time (WCET)

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-009 |
| **Requirement** | The AIQL shall complete all qualification checks for a single audio frame within 5 ms worst-case execution time on the target compute platform. |
| **ASIL** | B |
| **Rationale** | The 5 ms WCET budget is derived from the timing architecture (Section 4.3). Within the 50 ms frame period, 5 ms is allocated to AIQL qualification. Exceeding this budget would cause frame processing backlog and eventual loss of real-time qualification. WCET must be measured/analyzed on the target platform using static analysis or measurement-based methods per ISO 26262-6. |
| **Failure Mode Addressed** | Temporal FFI (AIQL itself failing to execute in time) |
| **Verification** | WCET analysis, HiL timing measurement |
| **Acceptance Criterion** | Measured WCET <= 5 ms on target platform under worst-case conditions (maximum audio frame size, all checks executed, cache cold start). |

### TSR-AIQL-010: ASIL B Development Process

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-010 |
| **Requirement** | The AIQL software shall be developed according to ISO 26262-6 ASIL B methods and work products, including: (a) Software safety requirements specification; (b) Software architectural design with freedom from interference; (c) Software unit design and implementation per ASIL B coding guidelines; (d) Software unit testing with MC/DC coverage >= 80%; (e) Software integration testing; (f) Software safety validation. |
| **ASIL** | B |
| **Rationale** | As the AIQL is the sole safety mechanism qualifying QM audio input, it must itself be developed to ASIL B integrity. The development process must produce the work products required by ISO 26262-6 Tables 1-9 for ASIL B. |
| **Failure Mode Addressed** | All (process-level requirement ensuring safety mechanism integrity) |
| **Verification** | Process audit, work product review |
| **Acceptance Criterion** | All ISO 26262-6 ASIL B work products completed and reviewed; MC/DC coverage >= 80% for all AIQL software units. |

### TSR-AIQL-011: Diagnostic Logging

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-011 |
| **Requirement** | The AIQL shall log all qualification state transitions, failure flag assertions/de-assertions, and recovery events to a non-volatile diagnostic log with timestamp. The log shall be accessible for post-drive analysis and shall retain data for at least 100 drive cycles. |
| **ASIL** | QM (diagnostic, not safety-critical) |
| **Rationale** | Diagnostic logging supports field monitoring, SOTIF validation, and continuous improvement of the siren classifier. Logging is QM because it does not contribute to the safety mechanism — it supports post-hoc analysis only. |
| **Failure Mode Addressed** | All (observability and field monitoring) |
| **Verification** | System test — verify log completeness and retention |
| **Acceptance Criterion** | All state transitions and failure events captured with timestamp resolution <= 1 ms; log retention verified over 100 drive cycles. |

### TSR-AIQL-012: Spatial FFI — Memory Protection

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-012 |
| **Requirement** | The AIQL code and data memory regions shall be protected by hardware MPU/MMU such that: (a) QM audio driver processes cannot write to AIQL code or data regions; (b) QM audio DMA is restricted to the designated shared audio buffer and cannot access AIQL memory; (c) Stack guard pages separate QM and AIQL memory partitions. |
| **ASIL** | B |
| **Rationale** | Spatial FFI ensures that a fault in the QM audio subsystem (buffer overflow, wild pointer, DMA descriptor corruption) cannot corrupt AIQL code or data, which would defeat the safety mechanism. Hardware enforcement (MPU/MMU) provides the highest level of isolation assurance. |
| **Failure Mode Addressed** | FM-08 (Common Cause), spatial FFI for all failure modes |
| **Verification** | FI-08 |
| **Acceptance Criterion** | 100% of unauthorized memory accesses from QM to AIQL memory blocked by MPU/MMU; AIQL state unchanged after attempted QM memory access violations. |

### TSR-AIQL-013: Temporal FFI — Watchdog and Scheduling

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-013 |
| **Requirement** | The AIQL execution shall be protected by: (a) A hardware watchdog with a timeout of 100 ms — if the AIQL fails to service the watchdog within this period, the watchdog shall force the AIQL output to NOT_QUALIFIED and trigger a system diagnostic event; (b) A guaranteed CPU execution budget provided by either a dedicated CPU core or a time-partitioned scheduler with minimum 10% CPU allocation for AIQL. |
| **ASIL** | B |
| **Rationale** | Temporal FFI ensures that a QM software failure (infinite loop, deadlock, priority inversion) cannot prevent the AIQL from executing. The 100 ms watchdog timeout is set to 2x the frame period (50 ms) to allow for legitimate scheduling jitter while catching genuine AIQL execution failures within the FTTI budget. |
| **Failure Mode Addressed** | FM-08 (Common Cause), temporal FFI for all failure modes |
| **Verification** | FI-09 |
| **Acceptance Criterion** | Watchdog triggers within 100 ms of AIQL execution starvation; NOT_QUALIFIED output asserted within 150 ms of starvation onset. |

### TSR-AIQL-015: BIST Reference Signal Generation (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-015 |
| **Requirement** | **Optional — active only when an exterior speaker is available on the vehicle platform.** The AIQL shall generate a logarithmic sine sweep reference signal from 500 Hz to 3000 Hz at -20 dBFS amplitude for acoustic loopback verification. The signal shall be output to the exterior speaker via the codec DAC interface (Section 4.2.5). The signal shall have total harmonic distortion (THD) < 1% and frequency accuracy within +/- 2% of the specified sweep profile. If no exterior speaker is present, this TSR is not applicable and BIST functionality is disabled. |
| **ASIL** | B (when active) |
| **Rationale** | The BIST reference signal must cover the siren frequency range (500-3000 Hz) to verify the audio path across all frequencies relevant to emergency vehicle detection. The -20 dBFS amplitude is chosen to be clearly detectable above typical ambient noise while remaining below levels that could interfere with exterior environment perception. Logarithmic sweep provides equal energy per octave, matching human hearing perception and siren frequency distribution. This TSR is optional because the Hyperion reference platform does not include an exterior speaker for BIST; passive qualification checks (TSR-AIQL-020, -021, -022) provide the primary diagnostic approach. |
| **Failure Mode Addressed** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) — both optional/customer-dependent |
| **Verification** | FI-10, unit test (when speaker is available) |
| **Acceptance Criterion** | Signal generation within specified frequency range, amplitude, and THD limits; sweep profile matches reference within +/- 2% frequency accuracy. |

### TSR-AIQL-016: BIST Spectral Match Verification (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-016 |
| **Requirement** | **Optional — active only when an exterior speaker is available on the vehicle platform.** The AIQL shall capture the microphone response to the BIST reference signal and perform spectral match verification. The verification shall compute: (a) Normalized cross-correlation between expected and received spectral envelopes — pass threshold: correlation >= 0.85; (b) Amplitude deviation per 1/3-octave band — pass threshold: deviation within +/- 6 dB of reference. Both criteria must be met for BIST pass. Failure of either criterion shall set the appropriate failure flag (FF_BIST_FAIL, FF_BIST_COUPLING). If no exterior speaker is present, this TSR is not applicable. |
| **ASIL** | B (when active) |
| **Rationale** | Spectral match verification confirms that the complete acoustic-to-digital path (speaker → air → microphone → codec → digital) preserves the frequency characteristics needed for siren detection. The 0.85 correlation threshold allows for normal acoustic variation while detecting significant path degradation. The +/- 6 dB amplitude tolerance per 1/3-octave band detects frequency-selective faults (e.g., microphone resonance shift, speaker cone damage) while accommodating acoustic transfer function variation. This TSR is optional because the Hyperion reference platform does not include an exterior speaker; passive qualification checks provide the primary diagnostic approach. |
| **Failure Mode Addressed** | FM-01 (Loss), FM-04 (Out-of-Range), FM-10 (Coupling Degradation) — FM-10 is optional/customer-dependent |
| **Verification** | FI-11, unit test (when speaker is available) |
| **Acceptance Criterion** | Correct pass/fail determination for all test vectors; zero false passes for injected path degradation exceeding thresholds; false failure rate < 0.1% under nominal conditions. |

### TSR-AIQL-017: Startup BIST Precondition (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-017 |
| **Requirement** | **Optional — active only when an exterior speaker is available on the vehicle platform.** The AIQL shall execute a full startup BIST (2000 ms sine sweep) before transitioning from NOT_QUALIFIED to QUALIFIED state on system initialization. The startup BIST must pass before any audio data is qualified for AMO. If the startup BIST fails, the AIQL shall retry up to 3 times. If all 3 retries fail, the AIQL shall remain in NOT_QUALIFIED state and request MRM transition per TSR-AIQL-019. If no exterior speaker is present, startup qualification relies on passive checks (TSR-AIQL-020, -021, -022) passing for a configurable number of consecutive frames. |
| **ASIL** | B (when active) |
| **Rationale** | Startup BIST ensures the audio path is verified before any audio data influences AMO sensor fusion. The 2000 ms sweep duration provides comprehensive frequency coverage. Three retries accommodate transient interference during vehicle startup. Total startup BIST budget: ~8.2 seconds worst case (4 attempts x 2050 ms). This TSR is optional because the Hyperion reference platform does not include an exterior speaker. |
| **Failure Mode Addressed** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) — both optional/customer-dependent |
| **Verification** | FI-12, unit test (when speaker is available) |
| **Acceptance Criterion** | No transition to QUALIFIED without BIST pass; all 3 retries executed on persistent failure; MRM requested after retry exhaustion. |

### TSR-AIQL-018: Periodic BIST Scheduling (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-018 |
| **Requirement** | **Optional — active only when an exterior speaker is available on the vehicle platform.** The AIQL shall execute a periodic BIST (200 ms abbreviated sweep burst) every 60 seconds during QUALIFIED operation. During the periodic BIST window, the AIQL shall: (a) Subtract the known BIST signal from the audio data before forwarding to AMO (signal cancellation); (b) Perform spectral match verification per TSR-AIQL-016 on the microphone capture. Signal cancellation shall achieve >= 30 dB attenuation of the BIST signal in the forwarded audio stream. If no exterior speaker is present, this TSR is not applicable; continuous passive monitoring (TSR-AIQL-020, -021, -022) provides ongoing diagnostic coverage without requiring a test signal. |
| **ASIL** | B (when active) |
| **Rationale** | Periodic BIST detects audio path degradation that develops during driving (e.g., microphone contamination, connector vibration loosening, thermal drift). The 60-second interval balances diagnostic coverage against environmental audio intrusion. The 200 ms burst is shorter than the startup sweep but covers the critical siren frequency range. Signal cancellation prevents the BIST signal from reaching AMO and causing false siren detections. The 30 dB cancellation target reduces the BIST signal below the classifier noise floor. This TSR is optional because the Hyperion reference platform does not include an exterior speaker. |
| **Failure Mode Addressed** | FM-10 (Coupling Degradation), FM-11 (BIST False Failure) — both optional/customer-dependent |
| **Verification** | FI-13, unit test (when speaker is available) |
| **Acceptance Criterion** | BIST executes within +/- 5 seconds of scheduled interval; signal cancellation >= 30 dB measured at AIQL output; no false siren detections by classifier during BIST window. |

### TSR-AIQL-019: MRM Transition on Persistent Qualification Failure

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-019 |
| **Requirement** | Upon persistent qualification failure, the AIQL shall: (a) Set qualification_status = NOT_QUALIFIED with applicable failure flags; (b) Transmit an MRM request to the vehicle safety manager via THOR FSI (SOC_Error / FSI_I2C safety signaling path); (c) Log the failure details and MRM request to the diagnostic log (TSR-AIQL-011). The MRM request shall trigger a controlled vehicle stop (Minimal Risk Maneuver). The MRM_REQUESTED state is terminal — recovery requires system restart. Persistent qualification failure is defined as: (i) Confirmed BIST failure (when BIST is enabled): startup 3 consecutive failures, periodic 2 consecutive failures — sets FF_BIST_FAIL; (ii) Persistent passive check failure: NOT_QUALIFIED state sustained for a configurable duration (default: 30 seconds) without recovery, indicating structural audio path degradation not recoverable by transient fault mechanisms. |
| **ASIL** | B |
| **Rationale** | MRM is appropriate when audio path degradation is persistent and cannot be resolved by software retry or transient recovery. For BIST-equipped platforms, BIST failure reveals structural degradation of the acoustic capture chain. For all platforms, sustained NOT_QUALIFIED state (passive checks persistently failing) indicates a hardware or environmental condition that precludes audio qualification. MRM ensures the vehicle reaches a safe state when audio path integrity cannot be verified. Two retries for periodic BIST (vs. 3 for startup) reflect the higher confidence that a failure during driving represents a real fault rather than transient noise. |
| **Failure Mode Addressed** | FM-09 (Speaker Failure — optional), FM-10 (Coupling Degradation — optional), FM-12 (Mic Blockage), FM-13 (Cross-Mic Plausibility Failure) |
| **Verification** | FI-14, unit test |
| **Acceptance Criterion** | MRM requested within 500 ms of confirmed persistent failure; no MRM request before retry/duration exhaustion; MRM_REQUESTED state is terminal (no automatic recovery). |

### TSR-AIQL-020: Noise Profile Monitoring

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-020 |
| **Requirement** | The AIQL shall monitor the noise profile of each exterior microphone channel every audio frame (20 Hz). The AIQL shall flag a channel as anomalous and set FF_NOISE_PROFILE if any of the following conditions are detected: (a) Silence — RMS level below noise floor threshold (configurable, default: 10 LSB RMS) indicating disconnected or blocked microphone; (b) DC rail — mean sample value within 500 LSB of positive or negative rail, indicating bias circuit failure; (c) Persistent saturation — more than 5% of samples clipped (|value| > 32000) for 3 or more consecutive frames, indicating gain fault or sustained overload; (d) Abnormal spectral content — spectral energy distribution deviates significantly from expected exterior noise profile (configurable spectral template), indicating hardware fault or severe environmental anomaly. |
| **ASIL** | B |
| **Rationale** | Noise profile monitoring provides continuous passive verification of microphone health without requiring a test signal or exterior speaker. A functioning exterior microphone in an operational vehicle always exhibits a characteristic noise profile (road noise, wind, ambient sound). Deviations from this profile indicate hardware faults (disconnected, blocked, bias failure) or environmental conditions (complete blockage by ice/mud) that compromise siren detection. This is the primary diagnostic mechanism for platforms without acoustic loopback BIST. |
| **Failure Mode Addressed** | FM-01 (Loss — silence detection), FM-04 (Out-of-Range — DC rail, saturation), FM-12 (Mic Blockage) |
| **Verification** | FI-15, unit test |
| **Acceptance Criterion** | 100% detection of silence, DC rail, and persistent saturation conditions within 3 frames (150 ms); abnormal spectral detection within 10 frames (500 ms); zero false alerts during nominal driving with ambient noise > 50 dB(A). |

### TSR-AIQL-021: Cross-Microphone Plausibility

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-021 |
| **Requirement** | The AIQL shall compare RMS levels and spectral characteristics across all exterior microphone channels every audio frame (20 Hz). The AIQL shall flag any channel deviating by more than X dB (configurable, default: 12 dB) from the median RMS of its peer channels and set FF_CROSSMIC. Cross-mic plausibility requires a minimum of 2 microphones. With fewer than 2 microphones, this check is disabled and a diagnostic log entry is generated at startup. |
| **ASIL** | B |
| **Rationale** | Cross-mic plausibility exploits the physical redundancy of the exterior microphone array. All exterior microphones on the same vehicle experience similar acoustic environments (road noise, wind, ambient sound). A single channel deviating significantly from its peers indicates a hardware fault (sensitivity loss, bias failure, wiring issue) or localized blockage (mud on one mic) specific to that channel. The median-based comparison is robust against a single outlier channel. The 12 dB default threshold accommodates normal variation due to microphone placement and directional sound sources while detecting faults that degrade siren detection capability. |
| **Failure Mode Addressed** | FM-01 (Loss — single channel), FM-04 (Out-of-Range — single channel), FM-12 (Mic Blockage — localized), FM-13 (Cross-Mic Plausibility Failure) |
| **Verification** | FI-16, unit test |
| **Acceptance Criterion** | Detection of single-channel divergence (>X dB from median) within 2 frames (100 ms); correct identification of the divergent channel; no false alerts for nominal multi-channel audio with directional sound sources. |

### TSR-AIQL-022: Input Sanitization

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-022 |
| **Requirement** | The AIQL shall verify each audio frame is within expected bounds before forwarding to AMO. Specifically: (a) Amplitude range — all samples within int16 range with no more than 5% clipped; (b) Noise floor — RMS above minimum threshold; (c) No persistent clipping — saturation must not persist for more than 3 consecutive frames; (d) No persistent DC offset — mean value within [-500, +500] LSB. Frames that fail any sanitization check shall be zero-filled (audio data and classifier output cleared), the qualification_status shall be set to NOT_QUALIFIED, and the frame shall be forwarded to AMO with FF_RANGE and/or FF_NOISE_PROFILE flags set. |
| **ASIL** | B |
| **Rationale** | Input sanitization is the final safety gate before audio data enters the AMO perception pipeline. It ensures that grossly abnormal audio (saturated, DC-railed, silent, or otherwise corrupted) does not reach AMO in a QUALIFIED state. By zero-filling and flagging non-conforming frames, the AIQL guarantees that AMO either receives valid audio or explicitly knows to exclude the audio channel from fusion. This complements the per-check failure flags (TSR-AIQL-001 through -005) with a holistic frame-level quality gate. |
| **Failure Mode Addressed** | FM-04 (Out-of-Range), FM-12 (Mic Blockage — attenuated/absent signal), FM-13 (Cross-Mic Plausibility Failure — divergent channel data) |
| **Verification** | FI-15, FI-17, unit test |
| **Acceptance Criterion** | 100% of out-of-bounds frames are zero-filled and flagged NOT_QUALIFIED before forwarding to AMO; zero leakage of grossly abnormal audio data to AMO in QUALIFIED state. |

### 7.1 TSR Summary Table

| TSR ID | Short Name | ASIL | Failure Modes | FFI Type | Status |
|--------|-----------|------|---------------|----------|--------|
| TSR-AIQL-001 | Freshness Check | B | FM-01, FM-03 | Communication | Mandatory |
| TSR-AIQL-002 | Alive Counter | B | FM-01, FM-03 | Communication | Mandatory |
| TSR-AIQL-003 | CRC-32 Integrity | B | FM-02 | Communication | Mandatory |
| TSR-AIQL-004 | Sequence Counter | B | FM-07 | Communication | Mandatory |
| TSR-AIQL-005 | Range Validation | B | FM-04 | Communication | Mandatory |
| TSR-AIQL-007 | Graceful Degradation | B | FM-01, FM-02, FM-03, FM-08, FM-09, FM-10, FM-12, FM-13 | — | Mandatory |
| TSR-AIQL-008 | Recovery Criteria | B | All | — | Mandatory |
| TSR-AIQL-009 | WCET | B | Temporal FFI | Temporal | Mandatory |
| TSR-AIQL-010 | ASIL B Process | B | All | — | Mandatory |
| TSR-AIQL-011 | Diagnostic Logging | QM | All | — | Mandatory |
| TSR-AIQL-012 | Spatial FFI (MPU) | B | FM-08, Spatial FFI | Spatial | Mandatory |
| TSR-AIQL-013 | Temporal FFI (Watchdog) | B | FM-08, Temporal FFI | Temporal | Mandatory |
| TSR-AIQL-015 | BIST Signal Generation | B | FM-09, FM-10 | — | **Optional** (customer-dependent) |
| TSR-AIQL-016 | BIST Spectral Match | B | FM-01, FM-04, FM-10 | — | **Optional** (customer-dependent) |
| TSR-AIQL-017 | Startup BIST | B | FM-09, FM-10 | — | **Optional** (customer-dependent) |
| TSR-AIQL-018 | Periodic BIST | B | FM-10, FM-11 | — | **Optional** (customer-dependent) |
| TSR-AIQL-019 | MRM on Persistent Qual. Failure | B | FM-09, FM-10, FM-12, FM-13 | — | Mandatory |
| TSR-AIQL-020 | Noise Profile Monitoring | B | FM-01, FM-04, FM-12 | — | Mandatory |
| TSR-AIQL-021 | Cross-Mic Plausibility | B | FM-01, FM-04, FM-12, FM-13 | — | Mandatory |
| TSR-AIQL-022 | Input Sanitization | B | FM-04, FM-12, FM-13 | — | Mandatory |

---

## 8. ASIL Decomposition

### 8.1 Decomposition Argument

The ASIL B safety integrity for SG-EMV-01 is achieved through ASIL decomposition per ISO 26262-9 Clause 5, applied at the boundary between the QM audio subsystem and the ASIL B AIQL.

```
Safety Goal SG-EMV-01: ASIL B
    |
    +--- ASIL B (AIQL on Thor Main, BP SW) — Safety mechanism qualifying audio input
    |     (passive checks mandatory; BIST optional/customer-dependent)
    |
    +--- QM (Audio HW) — Exterior microphones, codec, A2B transceiver
    |
    +--- QM (Audio SW) — Audio driver, DSP firmware, siren classifier
    |
    +--- QM (Speaker HW) — Exterior speaker, DAC, speaker amplifier (optional)
```

**Decomposition**: ASIL B = ASIL B(AIQL) + QM(Audio HW) + QM(Audio SW) + QM(Speaker HW, optional)

This decomposition is valid under ISO 26262-9 Clause 5 because:

1. **The ASIL B element (AIQL) implements all safety mechanisms necessary to detect and mitigate data integrity failures in the QM elements** — as enumerated in Section 7 (TSR-AIQL-001 through TSR-AIQL-005, TSR-AIQL-007 through TSR-AIQL-019). Classifier correctness (FM-05, FM-06) is addressed at the system level by AMO multi-sensor fusion with primary sensors, not by the AIQL.

2. **The QM elements are not required to perform any safety function** — they provide audio data with Assumptions of Use (Section 9) but their failure is handled by the AIQL

3. **Independence between the ASIL B and QM elements is ensured** — per Section 8.2 below

### 8.2 Independence Argument

Independence between the ASIL B AIQL and the QM audio subsystem is required per ISO 26262-9 for valid ASIL decomposition. The following arguments support independence:

#### 8.2.1 Hardware Independence

| Aspect | Evidence |
|--------|----------|
| **Separate processing** | AIQL executes on the NVIDIA SoC compute platform; audio codec is a separate IC on the audio board |
| **Separate power domains** | AIQL is powered from the SoC power rail; audio hardware is powered from the audio subsystem power rail (separate LDO) |
| **No shared single points of failure** | No single component whose failure affects both audio capture and AIQL qualification |
| **Speaker HW independence** | Speaker amplifier and DAC are separate ICs from the microphone codec ADC; speaker failure does not prevent AIQL from detecting microphone-side faults via passive checks (freshness, CRC, range) |

**Residual concern**: Shared SoC die for AIQL processing and audio DMA engine — mitigated by MPU/MMU spatial isolation (TSR-AIQL-012) and watchdog temporal isolation (TSR-AIQL-013).

#### 8.2.2 Software Independence

| Aspect | Evidence |
|--------|----------|
| **Separate development teams** | QM audio driver developed by audio team; AIQL developed by safety team per ASIL B process |
| **Separate codebases** | AIQL source code in separate repository with ASIL B configuration management |
| **No shared libraries** | AIQL uses only ASIL B qualified libraries; no shared code with QM audio driver |
| **Memory isolation** | MPU/MMU enforced separation (TSR-AIQL-012) |
| **Execution isolation** | Dedicated core or time partition (TSR-AIQL-013) |

#### 8.2.3 Common Cause Failure Analysis

| Common Cause | Mitigation | Residual Risk |
|-------------|------------|---------------|
| Shared SoC thermal event | Temperature monitoring with pre-emptive degradation; AIQL watchdog triggers safe state | Low |
| Shared clock source | AIQL uses independent watchdog timer for time-critical checks; plausibility against system clock | Low |
| Shared power supply (board-level) | Separate voltage monitoring for AIQL power rail; brownout detection triggers safe state | Low |
| EMI affecting both audio and SoC | Shielding and filtering per EMC design; AIQL CRC detects data corruption | Medium — addressed by CRC but not root cause |
| Systematic SW error in SoC BSP | AIQL uses minimal BSP footprint; safety-qualified BSP configuration | Low |
| Shared codec IC for ADC (mic) and DAC (speaker) | Codec failure detected by both passive checks (no audio frames) and BIST failure (no loopback); independent failure modes ensure at least one detection path | Medium — shared silicon; mitigated by dual detection paths |

### 8.3 Decomposition Compliance Summary

| ISO 26262-9 Requirement | Compliance |
|--------------------------|------------|
| ASIL decomposition scheme documented | Yes — Section 8.1 |
| Independence of decomposed elements argued | Yes — Section 8.2 |
| Common cause failure analysis performed | Yes — Section 8.2.3 |
| Sufficient safety mechanisms in higher-ASIL element | Yes — 20 TSRs (15 mandatory ASIL B + 5 optional ASIL B + 1 QM) covering all data integrity and diagnostic failure modes (Section 7); classifier correctness addressed at system level |
| QM element Assumptions of Use documented | Yes — Section 9 (14 AoUs: 12 mandatory + 2 optional) |

---

## 9. Assumptions of Use (AoU) for QM Audio Subsystem

The QM audio subsystem is treated as a Safety Element out of Context (SEooC) per ISO 26262-8 Clause 12. The following Assumptions of Use define the interface contract that the QM audio subsystem must fulfill for the AIQL to achieve ASIL B qualification.

**Note**: These AoUs do not elevate the audio subsystem to ASIL B. They define the minimum QM-level capabilities that the AIQL safety mechanisms rely upon. If an AoU is not met, the corresponding AIQL safety mechanism will detect the violation and transition to a degraded or safe state.

### AoU-001: CRC-32 Generation

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-001 |
| **Assumption** | The QM audio driver shall compute and append a CRC-32 checksum (polynomial 0x04C11DB7) over all audio frame fields prior to writing the frame to the shared memory buffer. |
| **Rationale** | TSR-AIQL-003 requires CRC-32 verification. The AIQL cannot generate a CRC over data it has not yet received — the QM driver must generate it at the source. |
| **TSR Dependency** | TSR-AIQL-003 |
| **Verification of AoU** | QM audio driver unit test; integration test with known CRC test vectors |

### AoU-002: Sequence Counter

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-002 |
| **Assumption** | The QM audio driver shall include a monotonically increasing 16-bit sequence counter in each audio frame, incremented by 1 for each new frame. |
| **Rationale** | TSR-AIQL-004 requires sequence counter validation. The AIQL uses this counter to detect replayed, reordered, and duplicated frames. |
| **TSR Dependency** | TSR-AIQL-004 |
| **Verification of AoU** | QM audio driver unit test; long-duration test for wraparound behavior |

### AoU-003: Timestamp

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-003 |
| **Assumption** | The QM audio driver shall include a capture timestamp (microsecond resolution, referenced to system monotonic clock) in each audio frame, representing the time of first sample capture for that frame. |
| **Rationale** | TSR-AIQL-001 requires freshness validation. The timestamp must accurately reflect capture time (not buffer write time) for freshness checks to be meaningful. |
| **TSR Dependency** | TSR-AIQL-001 |
| **Verification of AoU** | Timestamp accuracy test: measure jitter between timestamp and actual capture time; verify < 1 ms accuracy |

### AoU-004: Alive Counter

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-004 |
| **Assumption** | The QM audio driver shall include a modulo-256 alive counter in each audio frame, incremented by 1 for each new frame, starting from 0 at driver initialization. |
| **Rationale** | TSR-AIQL-002 requires alive counter validation. This provides a lightweight liveness indication independent of the sequence counter (which serves a different purpose — ordering vs. liveness). |
| **TSR Dependency** | TSR-AIQL-002 |
| **Verification of AoU** | QM audio driver unit test; stress test under high CPU load |

### AoU-005: Frame Rate

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-005 |
| **Assumption** | The QM audio subsystem shall deliver audio frames at a nominal rate of 20 Hz (50 ms period) with a maximum jitter of +/- 10 ms under all specified operating conditions. |
| **Rationale** | The AIQL timing architecture (Section 4.3) and FTTI budget (Section 5.3) are designed around a 20 Hz frame rate. Siren wail frequencies cycle at 1-3 Hz; 20 Hz provides >6x Nyquist for tracking siren presence transitions. The known frame rate is also used by recovery logic (TSR-AIQL-008) to count consecutive valid frames. |
| **TSR Dependency** | TSR-AIQL-001, TSR-AIQL-008, TSR-AIQL-009 |
| **Verification of AoU** | Frame rate measurement over 1-hour continuous operation; jitter histogram analysis |

### AoU-006: Memory Isolation Compatibility

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-006 |
| **Assumption** | The QM audio driver shall access only its designated shared memory buffer for audio frame output and shall not attempt to write to any other memory region. The driver shall be compatible with MPU/MMU memory protection configuration that restricts its write access. |
| **Rationale** | TSR-AIQL-012 requires spatial FFI via MPU/MMU. The QM driver must operate correctly within a memory-protected environment. |
| **TSR Dependency** | TSR-AIQL-012 |
| **Verification of AoU** | MPU/MMU fault injection test; static analysis for memory access violations |

### AoU-007: CPU Budget Compliance

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-007 |
| **Assumption** | The QM audio driver shall complete audio frame processing within its allocated CPU budget (specified during integration) and shall not exceed its time partition allocation or interfere with other scheduled tasks. |
| **Rationale** | TSR-AIQL-013 requires temporal FFI. The QM driver must not starve the AIQL of CPU resources. While the watchdog provides detection, AoU compliance prevents degraded mode entries due to driver misbehavior. |
| **TSR Dependency** | TSR-AIQL-013 |
| **Verification of AoU** | WCET analysis of QM audio driver; runtime monitoring under worst-case load |

### AoU-008: Signal Range

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-008 |
| **Assumption** | The QM audio subsystem shall output PCM audio samples as 16-bit signed integers (int16) and classifier outputs as IEEE 754 single-precision floating-point values (float32). All values shall be within the ranges specified in the interface definition (Section 4.2). |
| **Rationale** | TSR-AIQL-005 requires range validation. The data format must be well-defined for range checks to be meaningful. Format violations (NaN, infinity, type mismatch) are treated as data corruption. |
| **TSR Dependency** | TSR-AIQL-005 |
| **Verification of AoU** | Data format verification test; boundary value test at range limits |

### AoU-009: Exterior Microphone Specifications

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-009 |
| **Assumption** | The exterior microphone array shall consist of a minimum of 2 microphones (recommended 4+ for robust cross-mic plausibility coverage) mounted on the vehicle exterior with: (a) Sensitivity >= -38 dBV/Pa; (b) Signal-to-noise ratio >= 65 dB(A); (c) Frequency response covering 500 Hz to 3000 Hz (siren frequency range) with <= 3 dB variation; (d) Operating temperature range -40 C to +85 C; (e) Environmental protection rating IP67 or equivalent (dust-tight, protected against temporary immersion in water); (f) Placement on exterior body panels providing exposure to ambient acoustic environment for siren detection; (g) Mechanical protection against road debris, high-pressure wash, and ice accumulation. |
| **Rationale** | Exterior microphone placement provides direct exposure to external siren audio without cabin insulation attenuation (previously estimated at 20-30 dB loss for in-cabin placement). Exterior mounting requires robust environmental protection (IP67) to withstand road spray, rain, temperature extremes, and debris. Minimum 2 microphones enables cross-mic plausibility checking (TSR-AIQL-021); 4+ microphones recommended for robust spatial coverage and tolerance of individual channel blockage/failure. |
| **TSR Dependency** | TSR-AIQL-005 (noise floor check), TSR-AIQL-020 (noise profile monitoring), TSR-AIQL-021 (cross-mic plausibility), TSR-AIQL-022 (input sanitization) |
| **Verification of AoU** | Microphone specification review; exterior environmental qualification test (IP67, temperature, vibration); acoustic characterization at mounting positions; cross-mic plausibility validation with minimum mic count |

### AoU-010: Self-Diagnostic Reporting

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-010 |
| **Assumption** | The QM audio subsystem shall provide a self-diagnostic status byte in each audio frame indicating: (a) Microphone connectivity (all channels active); (b) Codec operational status; (c) Driver initialization status. A value of 0x00 indicates all-OK; non-zero values indicate specific faults per a defined encoding. |
| **Rationale** | While the AIQL performs independent qualification checks, QM self-diagnostic information provides an additional indicator for early fault detection and diagnostic logging (TSR-AIQL-011). This is a defense-in-depth measure — the AIQL does not rely solely on this information for safety decisions. |
| **TSR Dependency** | TSR-AIQL-011 (diagnostic logging) |
| **Verification of AoU** | Fault injection in QM subsystem; verify diagnostic byte correctly reports induced faults |

### AoU-011: Speaker Availability and Specifications (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-011 |
| **Assumption** | **Optional — applies only when an exterior speaker is available on the vehicle platform for acoustic loopback BIST.** If provided, the vehicle shall have at least one exterior speaker (e.g., backup warning speaker, AVAS/pedestrian alerting speaker) available for BIST signal output with: (a) Frequency response covering 500 Hz to 3000 Hz; (b) Sound pressure level >= 70 dB SPL at nearest exterior microphone position when driven at -20 dBFS; (c) Total harmonic distortion (THD) < 5% within the BIST frequency range; (d) Speaker accessible via codec DAC interface without requiring other vehicle systems to be active. The Hyperion reference platform does not include an exterior speaker for BIST purposes; this AoU is for customer platforms that choose to implement acoustic loopback. |
| **Rationale** | The acoustic loopback BIST requires a speaker capable of reproducing the sine sweep test signal across the siren frequency range. The 70 dB SPL minimum ensures adequate SNR for spectral match verification against typical ambient noise. The speaker must be independently addressable — BIST must function regardless of other vehicle system states. If no speaker is available, passive qualification checks (TSR-AIQL-020, -021, -022) provide the primary diagnostic approach. |
| **TSR Dependency** | TSR-AIQL-015 (BIST signal generation — optional), TSR-AIQL-016 (BIST spectral match — optional) |
| **Verification of AoU** | Speaker specification review; acoustic output measurement at exterior microphone positions; BIST signal reproduction quality test (when speaker is available) |

### AoU-012: Speaker-to-Microphone Acoustic Coupling (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-012 |
| **Assumption** | **Optional — applies only when an exterior speaker is available on the vehicle platform for acoustic loopback BIST.** The acoustic coupling between the BIST speaker and the exterior microphone array shall provide: (a) Signal-to-noise ratio >= 20 dB for the BIST signal at the nearest microphone position under worst-case ambient noise conditions (highway speed, wind noise); (b) Frequency response variation <= 10 dB across 500-3000 Hz range for the speaker-to-microphone acoustic path; (c) Stable acoustic coupling (variation < 3 dB) across operating temperature range and normal driving conditions. If no speaker is available, this AoU does not apply. |
| **Rationale** | The BIST spectral match algorithm (TSR-AIQL-016) requires sufficient SNR to distinguish the test signal from ambient noise. The 20 dB SNR minimum ensures reliable spectral correlation even in noisy environmental conditions. Frequency response and stability requirements ensure the BIST reference calibration remains valid across operating conditions. This AoU is optional because the Hyperion reference platform does not include an exterior speaker. |
| **TSR Dependency** | TSR-AIQL-016 (BIST spectral match — optional), TSR-AIQL-018 (periodic BIST — optional) |
| **Verification of AoU** | Exterior acoustic characterization; BIST SNR measurement across temperature and ambient noise operating points; long-term stability validation (when speaker is available) |

### AoU-013: A2B Frame Structure

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-013 |
| **Assumption** | A2B audio frames delivered to Thor Main shall include the following fields for data integrity verification: (a) CRC — per AoU-001; (b) Alive counter — per AoU-004; (c) Sequence counter — per AoU-002; (d) Timestamp — per AoU-003. The A2B transceiver (Maarva, supporting up to 16 channels) shall preserve these fields without modification during transport from the codec to the Thor Main SoC. |
| **Rationale** | The AIQL's data integrity checks (TSR-AIQL-001 through TSR-AIQL-004) depend on the A2B frame carrying integrity fields end-to-end from the audio codec/driver to the AIQL on Thor Main. If the A2B bus strips, reorders, or corrupts these fields, the AIQL cannot perform its safety function. This AoU consolidates the A2B-specific transport integrity assumptions in a single place. |
| **TSR Dependency** | TSR-AIQL-001 (Freshness), TSR-AIQL-002 (Alive Counter), TSR-AIQL-003 (CRC-32), TSR-AIQL-004 (Sequence Counter) |
| **Verification of AoU** | A2B frame structure specification review; integration test verifying end-to-end field integrity across A2B bus; confirm with systems engineering |

### AoU-014: Microphone Quantity for Cross-Mic Plausibility

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-014 |
| **Assumption** | The vehicle platform shall provide a minimum of 2 exterior microphones connected via the A2B path to enable cross-mic plausibility checking (TSR-AIQL-021). A minimum of 4 exterior microphones is recommended for robust cross-mic plausibility coverage (tolerance of single-channel failure without loss of plausibility checking capability). |
| **Rationale** | Cross-mic plausibility (TSR-AIQL-021) compares RMS levels and spectral characteristics across microphone channels to detect individual channel faults. With only 1 microphone, no cross-comparison is possible and this safety mechanism is disabled (the AIQL falls back to noise profile monitoring and input sanitization only). With 2 microphones, a divergence is detectable but the algorithm cannot determine which channel is faulty without additional information. With 4+ microphones, majority-vote logic can identify and isolate the faulty channel while maintaining plausibility checking on the remaining channels. |
| **TSR Dependency** | TSR-AIQL-021 (Cross-Mic Plausibility) |
| **Verification of AoU** | Platform configuration review; verify minimum 2 mic channels present in A2B frame; cross-mic plausibility functional test with minimum and recommended mic counts |

### 9.1 AoU Summary Table

| AoU ID | Short Name | TSR Dependency | Verification Method | Status |
|--------|-----------|----------------|---------------------|--------|
| AoU-001 | CRC-32 Generation | TSR-AIQL-003 | Unit test, integration test | Mandatory |
| AoU-002 | Sequence Counter | TSR-AIQL-004 | Unit test, wraparound test | Mandatory |
| AoU-003 | Timestamp | TSR-AIQL-001 | Accuracy measurement | Mandatory |
| AoU-004 | Alive Counter | TSR-AIQL-002 | Unit test, stress test | Mandatory |
| AoU-005 | Frame Rate | TSR-AIQL-001, -008, -009 | Long-duration measurement | Mandatory |
| AoU-006 | Memory Isolation | TSR-AIQL-012 | MPU fault injection, static analysis | Mandatory |
| AoU-007 | CPU Budget | TSR-AIQL-013 | WCET analysis, runtime monitoring | Mandatory |
| AoU-008 | Signal Range | TSR-AIQL-005 | Format verification, boundary values | Mandatory |
| AoU-009 | Exterior Microphone Specs | TSR-AIQL-005, -020, -021, -022 | Spec review, env. qualification, acoustic test | Mandatory |
| AoU-010 | Self-Diagnostic | TSR-AIQL-011 | Fault injection | Mandatory |
| AoU-011 | Speaker Availability | TSR-AIQL-015, -016 | Spec review, acoustic measurement | **Optional** (customer-dependent) |
| AoU-012 | Acoustic Coupling | TSR-AIQL-016, -018 | Acoustic characterization, stability test | **Optional** (customer-dependent) |
| AoU-013 | A2B Frame Structure | TSR-AIQL-001, -002, -003, -004 | Spec review, integration test | Mandatory |
| AoU-014 | Mic Quantity for Plausibility | TSR-AIQL-021 | Config review, functional test | Mandatory |

---

## 10. Safe State & Degradation Strategy

### 10.1 Qualification State Machine

The AIQL maintains a qualification state machine that governs the safety status of the audio input path.

```
                  All checks pass
                  for 10 consecutive frames
              +---------------------------+
              |                           |
              v                           |
     +================+          +================+
     |   QUALIFIED    |--------->|   DEGRADED     |
     |   (Normal)     |  Any     |   (Monitoring) |
     +================+  single  +================+
              ^           check          |
              |           fails          | 3 consecutive
              |                          | frame failures
              |                          | OR timeout
              |                          v
              |               +================+
              |               | NOT_QUALIFIED  |
              +---------------|   (Safe State) |
              20 consecutive  +================+
              valid frames           |
              (passive faults        | Persistent qualification failure:
               only)                 |  - Confirmed BIST failure (optional)
                                     |  - Sustained NOT_QUALIFIED > 30 s
                                     v
                          +====================+
                          |  MRM_REQUESTED     |
                          |  (Terminal State)  |
                          +====================+
                           No automatic recovery
                           Requires system restart
                           Safety status -> THOR FSI
```

### 10.2 State Definitions

#### QUALIFIED (Normal Operation)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | All checks passing (data integrity + passive monitoring); sufficient consecutive valid frames received (10 from DEGRADED, 20 from NOT_QUALIFIED); startup BIST passed (if BIST enabled) or passive checks stable (if BIST not available) |
| **Behavior** | Audio data and classifier output passed through to AMO with qualification_status = QUALIFIED |
| **AMO Action** | Multi-sensor fusion with audio as supplementary input for emergency vehicle detection (nominal mode) |
| **Output** | Full audio frame + classifier output + qualification_status = QUALIFIED |

#### DEGRADED (Monitoring)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | Any single qualification check failure while in QUALIFIED state |
| **Behavior** | Audio data still passed through to AMO with qualification_status = DEGRADED; failure_flags indicate which checks failed; AIQL monitors for recovery or further degradation |
| **AMO Action** | Reduced confidence in audio channel; increase weight on primary sensors (camera, LiDAR, radar) for emergency vehicle detection |
| **Output** | Full audio frame + classifier output + failure_flags + qualification_status = DEGRADED |
| **Escalation** | Transition to NOT_QUALIFIED after 3 consecutive frame failures OR frame reception timeout > 150 ms |

#### NOT_QUALIFIED (Safe State)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | 3 consecutive frame failures in DEGRADED state; frame reception timeout > 150 ms; watchdog trigger; MPU violation detected |
| **Behavior** | Audio data and classifier output zeroed out; qualification_status = NOT_QUALIFIED; continue 20 Hz output for liveness signaling |
| **AMO Action** | Primary-sensor-only emergency vehicle detection; audio excluded from fusion. Primary sensors (camera, LiDAR, radar) continue normal operation. |
| **Output** | Zero-filled audio + zero-filled classifier + all failure_flags set + qualification_status = NOT_QUALIFIED |
| **Recovery** | Transition to DEGRADED after 20 consecutive valid frames (1000 ms) — only for passive fault recovery. If NOT_QUALIFIED was caused by confirmed BIST failure, recovery is not possible — state transitions to MRM_REQUESTED. |

#### MRM_REQUESTED (Terminal State)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | Persistent qualification failure per TSR-AIQL-019: (i) Confirmed BIST failure (optional): startup 3 consecutive failures (TSR-AIQL-017), periodic 2 consecutive failures; (ii) Sustained NOT_QUALIFIED from passive checks for > 30 seconds (configurable), indicating structural audio path degradation |
| **Behavior** | AIQL outputs NOT_QUALIFIED status with applicable failure flags; safety status transmitted to THOR FSI via SOC_Error / FSI_I2C; diagnostic log captures failure details |
| **AMO Action** | Audio excluded from fusion; THOR FSI initiates controlled stop (Minimal Risk Maneuver) |
| **Output** | Zero-filled audio + zero-filled classifier + failure flags + qualification_status = NOT_QUALIFIED + safety status to THOR FSI |
| **Recovery** | **None** — MRM_REQUESTED is a terminal state. Recovery requires full system restart and successful qualification (BIST pass if speaker available, or sustained passive check pass). This reflects the persistent nature of the underlying hardware or environmental condition. |

### 10.3 Fallback and MRM Strategy

When the AIQL enters a degraded or failed state, two distinct strategies apply depending on the failure type:

| Aspect | Transient Fault (NOT_QUALIFIED) | Persistent Failure (MRM_REQUESTED) |
|--------|-------------------------------|------------------------------|
| **Trigger** | Data integrity or passive check failures (freshness, CRC, sequence, range, noise profile, cross-mic, timeout) | Confirmed BIST failure (if BIST enabled) after retry exhaustion; OR sustained NOT_QUALIFIED > 30 s from passive checks |
| **Failure Nature** | Potentially transient (software glitch, scheduling issue, transient EMI, brief blockage) | Likely persistent (hardware degradation, mic blockage by ice/mud, wiring fault, environmental damage) |
| **Vehicle Action** | Continue driving with primary sensors only; audio excluded from fusion | Controlled stop (MRM) via THOR FSI — vehicle brought to safe standstill |
| **Detection Sources** | Camera, LiDAR, radar continue nominal operation | Camera, LiDAR, radar continue during MRM maneuver |
| **Recovery** | Automatic — AIQL recovers when 20 consecutive valid frames received | Manual — requires system restart and successful qualification |
| **Justification** | Audio is a secondary sensor; transient loss is tolerable with primary sensors active | Persistent failure indicates structural audio path degradation; continued operation violates safety concept assumptions |
| **Duration** | Until recovery or end of drive cycle | Until vehicle stops and system is restarted |
| **Safety Signaling** | Qualification status in output frame to AMO | Safety status to THOR FSI via SOC_Error / FSI_I2C |
| **HMI Notification** | "Audio detection system degraded" | "Audio system fault — vehicle stopping" |

**Persistent Failure-to-MRM Timing Budget**:

BIST path (optional, customer-dependent):

| Phase | Duration | Running Total | Activity |
|-------|----------|---------------|----------|
| BIST failure detected | 0 ms | 0 ms | Periodic BIST spectral match fails |
| First retry | ~250 ms | 250 ms | Abbreviated sweep (200 ms) + analysis (50 ms) |
| Second retry | ~250 ms | 500 ms | Abbreviated sweep (200 ms) + analysis (50 ms) |
| Safety status to THOR FSI | <= 100 ms | 600 ms | AIQL sends MRM via SOC_Error / FSI_I2C |
| THOR FSI processing | <= 200 ms | 800 ms | MRM path planning initiated |
| Vehicle deceleration begins | <= 200 ms | 1000 ms | Brake application starts |
| Vehicle stop (from 60 km/h) | ~30000 ms | ~31000 ms | Controlled deceleration to standstill |
| **Total: BIST fault to vehicle stop** | | **~31 seconds** | |

Passive check path (all platforms):

| Phase | Duration | Running Total | Activity |
|-------|----------|---------------|----------|
| Passive check failure detected | 0 ms | 0 ms | Noise profile / cross-mic / sanitization fails |
| Transition to NOT_QUALIFIED | <= 200 ms | 200 ms | State machine transitions after consecutive failures |
| Sustained NOT_QUALIFIED timer | 30000 ms | 30200 ms | Configurable timeout without recovery |
| Safety status to THOR FSI | <= 100 ms | 30300 ms | AIQL sends MRM via SOC_Error / FSI_I2C |
| THOR FSI processing | <= 200 ms | 30500 ms | MRM path planning initiated |
| Vehicle deceleration begins | <= 200 ms | 30700 ms | Brake application starts |
| Vehicle stop (from 60 km/h) | ~30000 ms | ~60700 ms | Controlled deceleration to standstill |
| **Total: passive fault to vehicle stop** | | **~61 seconds** | |

### 10.4 FTTI Budget Allocation

The 500 ms FTTI is allocated across the degradation sequence:

| Phase | Duration | Running Total | Activity |
|-------|----------|---------------|----------|
| Fault occurrence | 0 ms | 0 ms | Audio path fault occurs |
| Fault detection | <= 100 ms | 100 ms | AIQL detects fault (2 frame periods) |
| State transition to DEGRADED | <= 50 ms | 150 ms | AIQL sets DEGRADED status |
| State transition to NOT_QUALIFIED | <= 50 ms | 200 ms | If fault persists, AIQL sets NOT_QUALIFIED |
| AMO fusion update | <= 100 ms | 300 ms | AMO excludes audio from fusion, continues with primary sensors |
| Vehicle response | <= 200 ms | 500 ms | Vehicle continues normal operation with primary sensors |

**Note**: The FTTI budget is conservative. In practice, the AIQL will detect most faults within a single frame period (50 ms), providing additional margin.

---

## 11. Verification & Validation

### 11.1 Test Levels

| Level | Environment | Scope | ISO 26262 Reference |
|-------|-------------|-------|---------------------|
| **Unit Test** | Host PC (x86/ARM cross-compile) | Individual AIQL check functions (freshness, CRC, range, etc.) | ISO 26262-6, Table 9 |
| **SiL Test** | Software-in-the-Loop simulator | AIQL state machine with simulated audio driver input | ISO 26262-6, Table 10 |
| **HiL Test** | Hardware-in-the-Loop with target SoC | AIQL on target platform with real audio hardware and injected faults | ISO 26262-4, Table 8 |
| **System Test** | Full vehicle or vehicle-representative bench | End-to-end audio path including microphones, codec, driver, AIQL, and AMO interface | ISO 26262-4, Table 9 |

### 11.2 Unit Test Requirements

| Test Area | Coverage Target | Method |
|-----------|----------------|--------|
| Freshness check logic | MC/DC >= 80% | Boundary value analysis: exact threshold, 1 ms below, 1 ms above |
| Alive counter validation | MC/DC >= 80% | All modular arithmetic edge cases (0→1, 254→255, 255→0) |
| CRC-32 computation | Statement 100% | Known CRC test vectors; single-bit error injection at every position |
| Sequence counter validation | MC/DC >= 80% | Monotonic, non-monotonic, wraparound, duplicate, gap scenarios |
| Range validation | MC/DC >= 80% | Boundary values for all 6 range checks; NaN/Inf injection |
| State machine | State/transition 100% | All 6 transitions (3 forward, 3 recovery); invalid transition attempts |
| Recovery logic | MC/DC >= 80% | Consecutive valid frame counting; interrupted recovery sequences |
| BIST signal generation | MC/DC >= 80% | Sweep frequency accuracy, amplitude accuracy, THD verification |
| BIST spectral match | MC/DC >= 80% | Correlation computation, threshold comparison, 1/3-octave band analysis |
| BIST state machine integration | State/transition 100% | Startup sequence, periodic scheduling, retry logic, MRM transition |
| BIST signal cancellation | MC/DC >= 80% | Cancellation depth measurement, residual signal analysis |
| BIST MRM request logic | MC/DC >= 80% | Retry counting, MRM request generation, terminal state enforcement |

### 11.3 Fault Injection Campaign

The following fault injection tests verify that the AIQL correctly detects and handles each identified failure mode. All tests are executed at the HiL level on the target compute platform.

#### FI-01: Audio Frame Loss / Staleness Injection

| Attribute | Value |
|-----------|-------|
| **ID** | FI-01 |
| **Objective** | Verify AIQL detects loss of audio frames and stale data |
| **Failure Modes** | FM-01 (Loss), FM-03 (Staleness) |
| **TSRs Verified** | TSR-AIQL-001 (Freshness), TSR-AIQL-002 (Alive Counter) |
| **Injection Method** | (a) Suppress audio frames from driver to AIQL for 100 ms, 200 ms, 500 ms durations; (b) Freeze timestamp while continuing to deliver frames; (c) Stop alive counter while continuing to deliver frames |
| **Pass Criteria** | AIQL transitions to DEGRADED within 100 ms of fault onset; transitions to NOT_QUALIFIED within 200 ms for sustained fault; recovery within specified consecutive frame count after fault removal |
| **Test Count** | 12 test cases (3 injection types x 4 durations) |

#### FI-02: Timestamp Manipulation

| Attribute | Value |
|-----------|-------|
| **ID** | FI-02 |
| **Objective** | Verify AIQL detects manipulated or incorrect timestamps |
| **Failure Modes** | FM-03 (Staleness) |
| **TSRs Verified** | TSR-AIQL-001 (Freshness) |
| **Injection Method** | (a) Offset timestamp by +200 ms (future); (b) Offset timestamp by -200 ms (past); (c) Set timestamp to 0; (d) Set timestamp to maximum uint64 value |
| **Pass Criteria** | All manipulated timestamps detected and frames rejected; no false rejections of correctly timestamped frames during recovery |
| **Test Count** | 8 test cases |

#### FI-03: Data Corruption Injection

| Attribute | Value |
|-----------|-------|
| **ID** | FI-03 |
| **Objective** | Verify AIQL CRC-32 detects data corruption |
| **Failure Modes** | FM-02 (Corruption) |
| **TSRs Verified** | TSR-AIQL-003 (CRC-32) |
| **Injection Method** | (a) Single-bit flip in audio data at random position; (b) Multi-bit corruption (burst error, 4-32 bits); (c) Corrupt CRC field itself; (d) Corrupt classifier output fields; (e) Corrupt frame header fields |
| **Pass Criteria** | 100% detection of all injected corruptions; no false CRC failures on uncorrupted frames (run 1 million uncorrupted frames to verify) |
| **Test Count** | 20 test cases (5 injection types x 4 corruption patterns) |

#### FI-04: Sequence Counter Manipulation

| Attribute | Value |
|-----------|-------|
| **ID** | FI-04 |
| **Objective** | Verify AIQL detects sequence counter anomalies |
| **Failure Modes** | FM-07 (Sequence Disorder) |
| **TSRs Verified** | TSR-AIQL-004 (Sequence Counter) |
| **Injection Method** | (a) Duplicate frame (same sequence number); (b) Reversed frame order (swap two consecutive frames); (c) Gap in sequence (skip 1, 5, 100 frames); (d) Counter reset to 0 mid-stream |
| **Pass Criteria** | All sequence anomalies detected within 1 frame; appropriate failure flag set |
| **Test Count** | 10 test cases |

#### FI-05: Signal Range Violation

| Attribute | Value |
|-----------|-------|
| **ID** | FI-05 |
| **Objective** | Verify AIQL detects out-of-range audio and classifier signals |
| **Failure Modes** | FM-04 (Out-of-Range) |
| **TSRs Verified** | TSR-AIQL-005 (Range Validation) |
| **Injection Method** | (a) Audio samples all at saturation (+32767 or -32768); (b) Audio samples all zero (silence/disconnected mic); (c) Large DC offset (mean > 1000 LSB); (d) Classifier probability > 1.0 or < 0.0; (e) Classifier probability = NaN or Inf; (f) Direction > 180 or < -180 degrees |
| **Pass Criteria** | All out-of-range conditions detected within 1 frame; appropriate failure flag set; no false range violations for signals within specified bounds |
| **Test Count** | 18 test cases (6 injection types x 3 severity levels) |

#### FI-07: State Machine Stress Test

| Attribute | Value |
|-----------|-------|
| **ID** | FI-07 |
| **Objective** | Verify AIQL state machine behavior under rapid fault onset/clearance patterns |
| **Failure Modes** | All |
| **TSRs Verified** | TSR-AIQL-007 (Graceful Degradation), TSR-AIQL-008 (Recovery Criteria) |
| **Injection Method** | (a) Rapid alternation between valid and invalid frames (every other frame for 10 seconds); (b) Simultaneous injection of multiple fault types; (c) Fault injection during recovery sequence (interrupt recovery); (d) Sustained fault for 60 seconds followed by recovery; (e) Maximum-duration NOT_QUALIFIED state followed by immediate valid data |
| **Pass Criteria** | No oscillation (> 2 transitions within 5 seconds) during intermittent faults; no recovery in fewer than specified consecutive valid frames; correct state transition timing; no undefined states or deadlocks |
| **Test Count** | 15 test cases |

#### FI-08: Spatial FFI — MPU Violation Injection

| Attribute | Value |
|-----------|-------|
| **ID** | FI-08 |
| **Objective** | Verify that MPU/MMU hardware protection prevents QM audio subsystem from corrupting AIQL memory |
| **Failure Modes** | FM-08 (Common Cause — spatial interference) |
| **TSRs Verified** | TSR-AIQL-012 (Spatial FFI — Memory Protection) |
| **Injection Method** | (a) QM test process attempts write to AIQL code region; (b) QM test process attempts write to AIQL data region; (c) QM DMA descriptor configured to target AIQL memory region; (d) QM stack overflow into AIQL guard page region |
| **Pass Criteria** | 100% of unauthorized memory accesses blocked by MPU/MMU; MPU fault exception raised; AIQL state and data unchanged after each injection; system diagnostic event logged |
| **Test Count** | 8 test cases (4 injection types x 2 memory regions) |

#### FI-09: Temporal FFI — CPU Starvation and Watchdog

| Attribute | Value |
|-----------|-------|
| **ID** | FI-09 |
| **Objective** | Verify that the hardware watchdog detects AIQL execution starvation and forces NOT_QUALIFIED output |
| **Failure Modes** | FM-08 (Common Cause — temporal interference) |
| **TSRs Verified** | TSR-AIQL-013 (Temporal FFI — Watchdog and Scheduling) |
| **Injection Method** | (a) Prevent AIQL task from being scheduled for 120 ms (exceeds 100 ms watchdog timeout); (b) QM audio driver consumes 100% CPU on AIQL core for 200 ms; (c) Interrupt storm on audio IRQ (10000 interrupts/sec for 500 ms); (d) AIQL task suspended mid-execution for 150 ms |
| **Pass Criteria** | Watchdog triggers within 100 ms of AIQL execution starvation; NOT_QUALIFIED output asserted within 150 ms of starvation onset; system diagnostic event logged; AIQL recovers after starvation removed |
| **Test Count** | 8 test cases (4 injection types x 2 severity levels) |

#### FI-10: Speaker Failure Injection (Optional, Customer-Dependent)

**Note**: FI-10 through FI-14 are only applicable when acoustic loopback BIST is enabled (exterior speaker available). For the Hyperion reference platform (no speaker), these tests are not required.

| Attribute | Value |
|-----------|-------|
| **ID** | FI-10 |
| **Objective** | Verify AIQL detects speaker subsystem failures preventing BIST execution |
| **Failure Modes** | FM-09 (Speaker Subsystem Failure — optional) |
| **TSRs Verified** | TSR-AIQL-015 (BIST Signal Generation — optional), TSR-AIQL-019 (MRM on Persistent Failure) |
| **Injection Method** | (a) Disconnect speaker output (open circuit); (b) Short speaker output; (c) Disable codec DAC; (d) Corrupt BIST signal output (inject noise) |
| **Pass Criteria** | FF_BIST_SPEAKER set within BIST timeout; MRM requested after retry exhaustion; no false speaker failure flags during normal operation |
| **Test Count** | 8 test cases (4 injection types x 2 BIST phases: startup, periodic) |

#### FI-11: Spectral Match Degradation (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | FI-11 |
| **Objective** | Verify AIQL detects acoustic coupling degradation via BIST spectral match |
| **Failure Modes** | FM-10 (Acoustic Coupling Degradation — optional) |
| **TSRs Verified** | TSR-AIQL-016 (BIST Spectral Match Verification — optional) |
| **Injection Method** | (a) Attenuate received signal by 10 dB, 20 dB, 30 dB (gradual mic sensitivity loss); (b) Notch filter at 1 kHz, 2 kHz (frequency-selective fault); (c) Add broadband noise at varying SNR levels; (d) Phase shift received signal (acoustic path change) |
| **Pass Criteria** | FF_BIST_COUPLING set when correlation < 0.85 or amplitude deviation > +/- 6 dB; correct pass for signals within tolerance; threshold behavior verified at boundary |
| **Test Count** | 12 test cases (4 injection types x 3 severity levels) |

#### FI-12: Startup BIST Blocking (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | FI-12 |
| **Objective** | Verify AIQL blocks QUALIFIED transition until startup BIST passes |
| **Failure Modes** | FM-09 (Speaker Failure — optional), FM-10 (Coupling Degradation — optional) |
| **TSRs Verified** | TSR-AIQL-017 (Startup BIST Precondition — optional) |
| **Injection Method** | (a) Fail startup BIST 1 time then pass (verify retry); (b) Fail startup BIST 2 times then pass; (c) Fail startup BIST 3 times (verify MRM request); (d) Inject transient noise during startup BIST |
| **Pass Criteria** | No QUALIFIED transition without BIST pass; correct retry count (up to 3); MRM requested after 3 consecutive failures; total startup time within budget |
| **Test Count** | 8 test cases (4 injection types x 2 failure patterns) |

#### FI-13: Periodic BIST During Driving (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | FI-13 |
| **Objective** | Verify periodic BIST execution and signal cancellation during QUALIFIED operation |
| **Failure Modes** | FM-10 (Coupling Degradation — optional), FM-11 (BIST False Failure — optional) |
| **TSRs Verified** | TSR-AIQL-018 (Periodic BIST Scheduling — optional) |
| **Injection Method** | (a) Inject environmental noise at various levels during periodic BIST; (b) Degrade acoustic coupling mid-drive; (c) Verify signal cancellation by monitoring AMO input during BIST; (d) Verify BIST interval timing under various system loads |
| **Pass Criteria** | Periodic BIST executes within +/- 5 seconds of 60-second interval; signal cancellation >= 30 dB; no false siren detections during BIST window; correct failure detection when coupling degrades |
| **Test Count** | 12 test cases (4 injection types x 3 operating conditions) |

#### FI-14: BIST-Triggered MRM (Optional, Customer-Dependent)

| Attribute | Value |
|-----------|-------|
| **ID** | FI-14 |
| **Objective** | Verify AIQL correctly requests MRM on confirmed BIST failure and enters terminal state |
| **Failure Modes** | FM-09 (Speaker Failure — optional), FM-10 (Coupling Degradation — optional) |
| **TSRs Verified** | TSR-AIQL-019 (MRM Transition on Persistent Qualification Failure) |
| **Injection Method** | (a) Persistent speaker failure during periodic BIST (2 retries then MRM); (b) Persistent coupling degradation during periodic BIST; (c) Verify MRM_REQUESTED is terminal (no recovery without restart); (d) Verify MRM request timing (within 500 ms of confirmed failure); (e) Verify MRM during startup BIST (3 retries then MRM) |
| **Pass Criteria** | MRM requested after exactly 2 periodic retries or 3 startup retries; MRM request within 500 ms; MRM_REQUESTED state is terminal; diagnostic log contains BIST failure details |
| **Test Count** | 10 test cases (5 injection types x 2 BIST phases) |

#### FI-15: Noise Profile Anomaly Injection

| Attribute | Value |
|-----------|-------|
| **ID** | FI-15 |
| **Objective** | Verify AIQL noise profile monitoring detects microphone health anomalies |
| **Failure Modes** | FM-01 (Loss — silence), FM-04 (Out-of-Range — DC rail, saturation), FM-12 (Mic Blockage) |
| **TSRs Verified** | TSR-AIQL-020 (Noise Profile Monitoring), TSR-AIQL-022 (Input Sanitization) |
| **Injection Method** | (a) Inject silence (zero samples) on individual channels; (b) Inject DC rail (samples near +32767 or -32768); (c) Inject persistent saturation (>10% of samples clipped for 5+ consecutive frames); (d) Inject abnormal spectral content (pure tone, white noise at unexpected level); (e) Inject gradually decreasing noise floor (simulating slow mic degradation) |
| **Pass Criteria** | FF_NOISE_PROFILE set within 3 frames (150 ms) for silence, DC rail, and persistent saturation; abnormal spectral detection within 10 frames (500 ms); correct channel identification; no false alerts during nominal driving with ambient noise > 50 dB(A); zero-fill and NOT_QUALIFIED for failed frames per TSR-AIQL-022 |
| **Test Count** | 10 test cases (5 injection types x 2 severity levels) |

#### FI-16: Cross-Mic Divergence Injection

| Attribute | Value |
|-----------|-------|
| **ID** | FI-16 |
| **Objective** | Verify AIQL cross-mic plausibility detects individual channel divergence from peers |
| **Failure Modes** | FM-12 (Mic Blockage — localized), FM-13 (Cross-Mic Plausibility Failure) |
| **TSRs Verified** | TSR-AIQL-021 (Cross-Microphone Plausibility) |
| **Injection Method** | (a) Attenuate single channel by 15 dB, 20 dB, 30 dB (simulating partial blockage or sensitivity loss); (b) Inject noise on single channel while peers are nominal; (c) Inject silence on single channel while peers are nominal; (d) Inject gradual divergence (1 dB/minute drift on one channel) |
| **Pass Criteria** | FF_CROSSMIC set within 2 frames (100 ms) for divergence exceeding threshold (default: 12 dB from median); correct identification of divergent channel; no false alerts for nominal multi-channel audio with directional sound sources or natural inter-channel variation |
| **Test Count** | 8 test cases (4 injection types x 2 channel configurations: 2-mic, 4-mic) |

#### FI-17: Exterior Mic Blockage Simulation

| Attribute | Value |
|-----------|-------|
| **ID** | FI-17 |
| **Objective** | Verify AIQL detects simulated exterior microphone blockage conditions |
| **Failure Modes** | FM-12 (Mic Blockage/Environmental Damage) |
| **TSRs Verified** | TSR-AIQL-020 (Noise Profile), TSR-AIQL-021 (Cross-Mic Plausibility), TSR-AIQL-022 (Input Sanitization) |
| **Injection Method** | (a) Attenuate signal by 20 dB on single channel (simulating mud/ice covering mic port); (b) Apply low-pass filter at 500 Hz on single channel (simulating water ingress altering frequency response); (c) Attenuate all channels simultaneously by 15 dB (simulating vehicle-wide environmental event, e.g., heavy ice accumulation); (d) Intermittent signal dropout on single channel (simulating loose connector due to vibration/thermal cycling) |
| **Pass Criteria** | Single-channel blockage detected by cross-mic plausibility within 2 frames; all-channel blockage detected by noise profile monitoring within 3 frames; correct state transition to DEGRADED (single channel) or NOT_QUALIFIED (all channels); intermittent dropout causes DEGRADED state with recovery when signal returns |
| **Test Count** | 6 test cases (4 injection types + 2 multi-channel scenarios) |

### 11.4 Fault Injection Campaign Summary

| FI ID | Failure Modes | TSRs Verified | Test Count | Level | Status |
|-------|---------------|---------------|------------|-------|--------|
| FI-01 | FM-01, FM-03 | TSR-AIQL-001, -002 | 12 | HiL | Mandatory |
| FI-02 | FM-03 | TSR-AIQL-001 | 8 | HiL | Mandatory |
| FI-03 | FM-02 | TSR-AIQL-003 | 20 | HiL | Mandatory |
| FI-04 | FM-07 | TSR-AIQL-004 | 10 | HiL | Mandatory |
| FI-05 | FM-04 | TSR-AIQL-005 | 18 | HiL | Mandatory |
| FI-07 | All (data integrity) | TSR-AIQL-007, -008 | 15 | HiL | Mandatory |
| FI-08 | FM-08 | TSR-AIQL-012 | 8 | HiL | Mandatory |
| FI-09 | FM-08 | TSR-AIQL-013 | 8 | HiL | Mandatory |
| FI-10 | FM-09 | TSR-AIQL-015, -019 | 8 | HiL | **Optional** (customer-dependent) |
| FI-11 | FM-10 | TSR-AIQL-016 | 12 | HiL | **Optional** (customer-dependent) |
| FI-12 | FM-09, FM-10 | TSR-AIQL-017 | 8 | HiL | **Optional** (customer-dependent) |
| FI-13 | FM-10, FM-11 | TSR-AIQL-018 | 12 | HiL | **Optional** (customer-dependent) |
| FI-14 | FM-09, FM-10 | TSR-AIQL-019 | 10 | HiL | **Optional** (customer-dependent) |
| FI-15 | FM-01, FM-04, FM-12 | TSR-AIQL-020, -022 | 10 | HiL | Mandatory |
| FI-16 | FM-12, FM-13 | TSR-AIQL-021 | 8 | HiL | Mandatory |
| FI-17 | FM-12 | TSR-AIQL-020, -021, -022 | 6 | HiL | Mandatory |
| **Total (mandatory)** | | | **123** | | |
| **Total (with optional BIST)** | | | **173** | | |

**Note**: FM-05 (False Negative) and FM-06 (False Positive) are classifier performance limitations addressed at the system level by AMO multi-sensor fusion with primary sensors. They are not covered by the AIQL fault injection campaign. FI-10 through FI-14 are only required for customer platforms that implement acoustic loopback BIST (exterior speaker available). FI-15 through FI-17 cover the new passive qualification checks (noise profile, cross-mic plausibility, exterior mic blockage) that serve as the primary diagnostic approach for all platforms.

### 11.5 ASIL B Process Compliance Work Products

The following work products are required per ISO 26262-6 for ASIL B software development:

| Work Product | ISO 26262 Reference | Status |
|-------------|---------------------|--------|
| Software safety requirements specification | Part 6, Clause 6 | Planned |
| Software architectural design | Part 6, Clause 7 | Planned |
| Software unit design and implementation | Part 6, Clause 8 | Planned |
| Software unit verification report | Part 6, Clause 9 | Planned |
| Software integration and verification report | Part 6, Clause 10 | Planned |
| Software safety validation report | Part 6, Clause 11 | Planned |
| Configuration management plan | Part 8, Clause 7 | Planned |
| Change management process | Part 8, Clause 8 | Planned |
| Verification review report | Part 8, Clause 9 | Planned |

---

## 12. SOTIF Considerations

The Safety of the Intended Functionality (ISO 21448) analysis identifies performance limitations and triggering conditions specific to audio-based emergency vehicle detection that may cause hazardous behavior even when the system is fault-free.

### TC-AUDIO-01: Ambient Noise Masking

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | High ambient noise environments (construction zones, heavy traffic, tunnels) mask siren audio below classifier detection threshold |
| **Functional Insufficiency** | QM siren classifier signal-to-noise ratio insufficient to detect siren in noise levels > 90 dB(A) |
| **Hazardous Behavior** | Loss of siren detection reduces AMO confidence in EMV classification — slower or missed identification of stationary/slow EMV in front on highway (SG-EMV-01) |
| **Scenario** | Urban construction zone with jackhammer noise at 100 dB(A); ambulance approaching from 200 m with siren at 123 dB(A) at 1 m (attenuated to ~70 dB(A) at 200 m); siren masked by construction noise |
| **Risk Reduction** | **System-level mitigation**: Primary sensors (camera, LiDAR, radar) provide independent emergency vehicle detection via flashing lights, vehicle shape, and motion patterns. Audio masking does not affect primary sensor capability. ODD consideration: reduced audio detection range in high-noise environments is documented as a known limitation of the secondary sensor. |
| **AIQL Response** | AIQL remains in QUALIFIED state — the audio I/O path is functioning correctly (data integrity is intact). The detection limitation is a classifier performance issue, not a data integrity failure. AIQL's range validation (TSR-AIQL-005) may detect if noise causes sustained saturation. |
| **Residual Risk** | Low — audio is a secondary sensor. Primary sensors maintain full emergency vehicle detection capability regardless of ambient noise. |

### TC-AUDIO-02: Siren-Like Environmental Sounds

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | Environmental sounds with frequency characteristics similar to emergency sirens (e.g., construction warning tones, musical vehicle horns, certain industrial machinery, car alarms) |
| **Functional Insufficiency** | QM siren classifier cannot reliably distinguish all siren-like sounds from actual emergency vehicle sirens |
| **Hazardous Behavior** | False positive siren detection leading to unwarranted yielding maneuver — availability impact |
| **Scenario** | Vehicle passing construction site with reversing alarm (1000 Hz pulsing tone); classifier reports siren probability 0.75; no emergency vehicle present |
| **Risk Reduction** | **System-level mitigation**: AMO multi-sensor fusion weighs audio as a secondary input. A false siren detection from audio alone, without corroborating primary sensor evidence (flashing lights on camera, emergency vehicle shape on LiDAR), will be downweighted by fusion and will not trigger a yielding maneuver. |
| **AIQL Response** | AIQL remains in QUALIFIED state — the audio I/O path is functioning correctly. The classifier output passes all range checks (probability within [0.0, 1.0]). False positive detection is a classifier performance issue addressed at the system level. |
| **Residual Risk** | Low — audio is a secondary sensor. False audio-only siren detections do not independently trigger yielding; primary sensor corroboration is required by AMO's fusion architecture. |

### TC-AUDIO-03: Doppler Shift

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | Emergency vehicle approaching at high relative speed causes Doppler shift that moves siren fundamental frequency outside the classifier's trained frequency band |
| **Functional Insufficiency** | QM siren classifier trained on stationary siren frequency profiles; Doppler-shifted frequencies may fall outside recognition band |
| **Hazardous Behavior** | Delayed or missed siren detection when emergency vehicle approaches at high closing speed — potential SG-EMV-01 violation |
| **Scenario** | Emergency vehicle approaching head-on at 120 km/h relative speed; siren fundamental at 1000 Hz shifted to ~1100 Hz; if classifier frequency band is narrowly tuned, shifted frequency may reduce detection confidence |
| **Risk Reduction** | **System-level mitigation**: Primary sensors (camera, LiDAR, radar) detect emergency vehicles by visual/shape/motion characteristics unaffected by Doppler shift. Classifier AoU should specify Doppler-robust frequency band (siren wail sweeps 500-1600 Hz; Doppler shift at max relative speed is ~10%). SOTIF validation testing with Doppler-shifted audio. |
| **AIQL Response** | AIQL cannot detect Doppler-related misclassification — this is a classifier performance limitation, not a data integrity failure. Diagnostic logging (TSR-AIQL-011) captures events for post-drive analysis. |
| **Residual Risk** | Low — audio is a secondary sensor. Doppler shift at typical urban speeds (< 60 km/h relative) is < 5%, well within siren wail sweep range. Primary sensors unaffected by acoustic Doppler. |

### TC-AUDIO-04: Exterior Microphone Environmental Exposure

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | Exterior microphones are exposed to wind noise, road spray, debris impact, temperature extremes, and vehicle-speed-dependent aerodynamic noise that may mask or distort external siren audio |
| **Functional Insufficiency** | Exterior microphone placement provides direct siren exposure but introduces environmental noise sources absent with in-cabin mounting: wind noise increases with vehicle speed (typically 60-80 dB(A) at highway speed); road spray and debris may temporarily attenuate or block microphone ports; temperature extremes may affect microphone sensitivity |
| **Hazardous Behavior** | Reduced siren detection range at high vehicle speeds due to wind/road noise masking; temporary loss of siren detection during adverse weather (heavy rain, ice accumulation, road spray). Siren detection range affected by vehicle speed and ambient noise. |
| **Scenario** | Highway driving at 130 km/h with rain; wind noise at 75 dB(A) at microphone positions; road spray intermittently coating mic ports; exterior siren at 200 m (siren source at 123 dB(A) at 1 m, attenuated to ~70 dB(A) at 200 m in free field); wind noise reduces effective siren SNR to ~-5 dB at 200 m — detection range reduced until siren is closer |
| **Risk Reduction** | **System-level mitigation**: Primary sensors (camera, LiDAR, radar) detect emergency vehicles by visual/shape/motion characteristics unaffected by acoustic environmental exposure. AoU-009 specifies IP67 environmental protection for exterior mics. Noise profile monitoring (TSR-AIQL-020) detects blocked or degraded microphones. Cross-mic plausibility (TSR-AIQL-021) identifies individual mic channel degradation. Exterior placement eliminates the 20-30 dB cabin insulation attenuation that was the primary limitation of in-cabin placement (TC-AUDIO-04 v0.8), significantly improving effective siren detection range in normal conditions. |
| **AIQL Response** | Noise profile monitoring (TSR-AIQL-020) detects if environmental conditions cause mic anomalies (silence from blockage, saturation from wind gust). Cross-mic plausibility (TSR-AIQL-021) detects if individual mics are affected while others remain functional. Input sanitization (TSR-AIQL-022) prevents environmentally degraded audio from reaching AMO in a QUALIFIED state. AIQL transitions to DEGRADED or NOT_QUALIFIED if persistent environmental degradation is detected. |
| **Residual Risk** | **Low-Medium** — exterior placement significantly improves siren detection range vs. in-cabin placement (elimination of 20-30 dB cabin attenuation). However, high-speed wind noise introduces a speed-dependent detection range reduction. Detection range at 130 km/h estimated at ~100-150 m (vs. ~60-100 m for in-cabin at any speed). Primary sensors maintain full emergency vehicle detection capability regardless of acoustic conditions. The trade-off (exterior exposure vs. direct acoustic access) is favorable for siren detection performance. |

### TC-AUDIO-05: Acoustic Loopback Environmental Noise Interference (Customer-Dependent)

**Note**: This triggering condition only applies when acoustic loopback BIST is enabled (exterior speaker available on the vehicle platform). If no speaker is present, BIST is not executed and this TC does not apply. For platforms without BIST, continuous passive monitoring (noise profile, cross-mic plausibility) provides ongoing diagnostic coverage without injecting a test signal, avoiding this category of false failure entirely.

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | High environmental noise during BIST window (wind noise at speed, traffic noise, construction, heavy rain) masks the BIST reference signal at exterior microphones |
| **Functional Insufficiency** | BIST spectral match algorithm cannot reliably distinguish BIST signal from high-level environmental noise when SNR < 10 dB |
| **Hazardous Behavior** | False BIST failure triggers unnecessary MRM — availability impact (vehicle performs controlled stop when audio path is actually functional) |
| **Scenario** | Periodic BIST executes during highway driving at 120 km/h; wind noise at 75 dB(A) at exterior mic positions; BIST signal at -20 dBFS produces ~70 dB SPL at nearest microphone; wind energy in BIST frequency range masks the sweep signal; spectral correlation drops below 0.85 threshold |
| **Risk Reduction** | Retry mechanism (TSR-AIQL-019) — 2 retries for periodic BIST reduce false failure probability (noise conditions likely to change between retries). BIST signal level (-20 dBFS) designed to exceed typical ambient noise at microphone positions. Future enhancement: BIST scheduling could avoid windows with detected high ambient noise (see OI-14). Signal cancellation technique (TSR-AIQL-018) partially mitigates by subtracting known BIST signal before spectral analysis. |
| **AIQL Response** | If noise causes BIST failure, retry mechanism provides second chance. If both retries fail, MRM is requested — this is a conservative (safe) response. The false MRM rate is an availability concern, not a safety concern. |
| **Residual Risk** | Low for safety (false MRM is safe, not hazardous). **Medium for availability** — false BIST failures reduce system uptime. Target: false BIST failure rate < 1 per 1000 driving hours (see OI-11). This risk only applies to customer platforms that implement acoustic loopback; the Hyperion reference platform avoids this risk entirely by relying on passive qualification checks. |

### 12.1 SOTIF Summary

| TC ID | Triggering Condition | Primary Mitigation | AIQL Mechanism | Residual Risk |
|-------|---------------------|-------------------|----------------|---------------|
| TC-AUDIO-01 | Ambient noise masking | Primary sensors (system-level) | TSR-AIQL-005 (saturation detection), TSR-AIQL-020 (noise profile) | Low |
| TC-AUDIO-02 | Siren-like sounds | Primary sensors + fusion weighting (system-level) | None (classifier performance) | Low |
| TC-AUDIO-03 | Doppler shift | Primary sensors (system-level) | None (classifier performance) | Low |
| TC-AUDIO-04 | Exterior mic environmental exposure | Primary sensors + exterior mic specs (IP67) | AoU-009, TSR-AIQL-020 (noise profile), TSR-AIQL-021 (cross-mic) | Low-Medium |
| TC-AUDIO-05 | Acoustic loopback env. noise interference | Retry mechanism + signal level design | TSR-AIQL-019 (retries), TSR-AIQL-018 (cancellation) | Low (safety) / Medium (availability) — **customer-dependent** |

**Key insight**: Because audio is a secondary sensor, SOTIF triggering conditions TC-AUDIO-01 through TC-AUDIO-03 have **low residual risk** — primary sensors (camera, LiDAR, radar) maintain full emergency vehicle detection capability regardless of audio performance limitations. TC-AUDIO-04 is updated to reflect exterior microphone placement (v0.9): exterior mounting eliminates the 20-30 dB cabin insulation attenuation that was the primary limitation in previous versions, significantly improving effective siren detection range. The residual risk is reduced from Medium to **Low-Medium** because the primary concern shifts from cabin attenuation (inherent, unavoidable) to speed-dependent wind noise (mitigable by mic shielding design and detectable by noise profile monitoring). TC-AUDIO-05 is now **customer-dependent** — it only applies when acoustic loopback BIST is enabled with an exterior speaker. For the Hyperion reference platform (no exterior speaker), this TC does not apply; passive qualification checks avoid BIST false failure entirely. The AIQL's scope remains limited to data integrity qualification and passive/active path verification at the I/O boundary; classifier performance limitations are addressed at the system level by AMO multi-sensor fusion architecture.

---

## 13. Traceability Matrix

### 13.1 Safety Goal → Failure Mode → TSR Traceability

| Safety Goal | Failure Mode | TSR(s) | Verification | Status |
|------------|-------------|--------|-------------|--------|
| SG-EMV-01 | FM-01: Complete Loss | TSR-AIQL-001, -002, -007, -016 (opt), -020, -021 | FI-01, FI-15 | Mandatory |
| SG-EMV-01 | FM-02: Corruption | TSR-AIQL-003, -005, -007 | FI-03 | Mandatory |
| SG-EMV-01 | FM-03: Staleness | TSR-AIQL-001, -002, -004, -007 | FI-01, FI-02 | Mandatory |
| SG-EMV-01 | FM-04: Out-of-Range | TSR-AIQL-005, -016 (opt), -020, -021, -022 | FI-05, FI-15, FI-16 | Mandatory |
| SG-EMV-01 | FM-05: False Negative | — (system-level: AMO multi-sensor fusion) | — (system-level) | — |
| SG-EMV-01 | FM-06: False Positive | — (system-level: AMO multi-sensor fusion) | — (system-level) | — |
| SG-EMV-01 | FM-07: Sequence Disorder | TSR-AIQL-004 | FI-04 | Mandatory |
| SG-EMV-01 | FM-08: Common Cause | TSR-AIQL-012, -013, -007 | FI-07 | Mandatory |
| SG-EMV-01 | FM-09: Speaker Failure (opt) | TSR-AIQL-015, -019, -007 | FI-10 | Optional |
| SG-EMV-01 | FM-10: Coupling Degradation (opt) | TSR-AIQL-016, -019, -007 | FI-11 | Optional |
| SG-EMV-01 | FM-11: BIST False Failure (opt) | TSR-AIQL-018, -019 | FI-13 | Optional |
| SG-EMV-01 | FM-12: Mic Blockage/Env. Damage | TSR-AIQL-020, -021, -022, -019 | FI-15, FI-16, FI-17 | Mandatory |
| SG-EMV-01 | FM-13: Cross-Mic Plausibility | TSR-AIQL-021, -022, -019 | FI-16 | Mandatory |

### 13.2 TSR → AoU Traceability

| TSR | AoU Dependency | Rationale |
|-----|---------------|-----------|
| TSR-AIQL-001 (Freshness) | AoU-003 (Timestamp), AoU-005 (Frame Rate) | Freshness check requires accurate timestamps at known rate |
| TSR-AIQL-002 (Alive Counter) | AoU-004 (Alive Counter) | AIQL validates alive counter that QM driver generates |
| TSR-AIQL-003 (CRC-32) | AoU-001 (CRC Generation) | AIQL verifies CRC that QM driver computes |
| TSR-AIQL-004 (Sequence Counter) | AoU-002 (Sequence Counter) | AIQL validates sequence that QM driver generates |
| TSR-AIQL-005 (Range) | AoU-008 (Signal Range), AoU-009 (Mic Specs) | Range checks depend on defined data format and adequate hardware |
| TSR-AIQL-007 (Degradation) | — | AIQL internal behavior; no AoU dependency |
| TSR-AIQL-008 (Recovery) | AoU-005 (Frame Rate) | Recovery counting depends on known frame rate |
| TSR-AIQL-009 (WCET) | AoU-007 (CPU Budget) | AIQL WCET must fit within total timing budget |
| TSR-AIQL-010 (Process) | — | Process requirement; no AoU dependency |
| TSR-AIQL-011 (Logging) | AoU-010 (Self-Diagnostic) | Diagnostic logging includes QM self-diagnostic data |
| TSR-AIQL-012 (Spatial FFI) | AoU-006 (Memory Isolation) | QM driver must be compatible with MPU configuration |
| TSR-AIQL-013 (Temporal FFI) | AoU-007 (CPU Budget) | QM driver must comply with CPU budget for FFI |
| TSR-AIQL-015 (BIST Signal Gen, opt) | AoU-011 (Speaker Availability) | BIST requires speaker with specified frequency range and SPL |
| TSR-AIQL-016 (BIST Spectral Match, opt) | AoU-009 (Exterior Mic Specs), AoU-012 (Acoustic Coupling) | Spectral match depends on mic placement and acoustic coupling |
| TSR-AIQL-017 (Startup BIST, opt) | AoU-011 (Speaker Availability), AoU-012 (Acoustic Coupling) | Startup BIST requires functional speaker and adequate coupling |
| TSR-AIQL-018 (Periodic BIST, opt) | AoU-011 (Speaker Availability), AoU-012 (Acoustic Coupling) | Periodic BIST requires maintained acoustic path |
| TSR-AIQL-019 (MRM on Persistent Failure) | — | AIQL internal behavior; no AoU dependency |
| TSR-AIQL-020 (Noise Profile) | AoU-009 (Exterior Mic Specs), AoU-013 (A2B Frame Structure) | Noise profile analysis requires functioning exterior mics and known frame format |
| TSR-AIQL-021 (Cross-Mic Plausibility) | AoU-009 (Exterior Mic Specs), AoU-014 (Mic Quantity) | Cross-mic comparison requires >= 2 functioning exterior mics |
| TSR-AIQL-022 (Input Sanitization) | AoU-008 (Signal Range), AoU-009 (Exterior Mic Specs) | Sanitization checks depend on defined data format and expected mic behavior |

### 13.3 Failure Mode Coverage Analysis

Every failure mode must be addressed by at least one TSR. Every TSR must address at least one failure mode.

| FM | TSR Coverage Count | TSR IDs | Status |
|----|-------------------|---------|--------|
| FM-01 | 6 | TSR-AIQL-001, -002, -007, -016 (opt), -020, -021 | Mandatory |
| FM-02 | 3 | TSR-AIQL-003, -005, -007 | Mandatory |
| FM-03 | 4 | TSR-AIQL-001, -002, -004, -007 | Mandatory |
| FM-04 | 5 | TSR-AIQL-005, -016 (opt), -020, -021, -022 | Mandatory |
| FM-05 | 0* | — (system-level: AMO multi-sensor fusion) | — |
| FM-06 | 0* | — (system-level: AMO multi-sensor fusion) | — |
| FM-07 | 1 | TSR-AIQL-004 | Mandatory |
| FM-08 | 3 | TSR-AIQL-007, -012, -013 | Mandatory |
| FM-09 | 3 | TSR-AIQL-007, -015, -019 | Optional |
| FM-10 | 3 | TSR-AIQL-007, -016, -019 | Optional |
| FM-11 | 2 | TSR-AIQL-018, -019 | Optional |
| FM-12 | 4 | TSR-AIQL-020, -021, -022, -019 | Mandatory |
| FM-13 | 3 | TSR-AIQL-021, -022, -019 | Mandatory |

*FM-05 and FM-06 are classifier performance limitations (not data integrity failures). As audio is a secondary sensor, these are addressed at the system level by AMO multi-sensor fusion with primary sensors (camera, LiDAR, radar) — not by the AIQL.

**Coverage check**: All 11 data integrity and diagnostic failure modes (FM-01 through FM-04, FM-07 through FM-13) have at least 1 AIQL TSR. FM-05 and FM-06 (classifier performance) are addressed at system level. FM-09 through FM-11 (BIST-related) are optional/customer-dependent. **PASS**

| TSR | FM Coverage Count | FM IDs | Status |
|-----|-------------------|--------|--------|
| TSR-AIQL-001 | 2 | FM-01, FM-03 | Mandatory |
| TSR-AIQL-002 | 2 | FM-01, FM-03 | Mandatory |
| TSR-AIQL-003 | 1 | FM-02 | Mandatory |
| TSR-AIQL-004 | 2 | FM-03, FM-07 | Mandatory |
| TSR-AIQL-005 | 2 | FM-02, FM-04 | Mandatory |
| TSR-AIQL-007 | 6 | FM-01, FM-02, FM-03, FM-08, FM-09, FM-10 | Mandatory |
| TSR-AIQL-008 | 13 | All (recovery behavior) | Mandatory |
| TSR-AIQL-009 | 0* | Temporal FFI (AIQL self-protection) | Mandatory |
| TSR-AIQL-010 | 0* | Process requirement (all FMs) | Mandatory |
| TSR-AIQL-011 | 0* | Diagnostic (all FMs, observability) | Mandatory (QM) |
| TSR-AIQL-012 | 1 | FM-08 | Mandatory |
| TSR-AIQL-013 | 1 | FM-08 | Mandatory |
| TSR-AIQL-015 | 2 | FM-09, FM-10 | Optional |
| TSR-AIQL-016 | 3 | FM-01, FM-04, FM-10 | Optional |
| TSR-AIQL-017 | 2 | FM-09, FM-10 | Optional |
| TSR-AIQL-018 | 2 | FM-10, FM-11 | Optional |
| TSR-AIQL-019 | 4 | FM-09, FM-10, FM-12, FM-13 | Mandatory |
| TSR-AIQL-020 | 3 | FM-01, FM-04, FM-12 | Mandatory |
| TSR-AIQL-021 | 4 | FM-01, FM-04, FM-12, FM-13 | Mandatory |
| TSR-AIQL-022 | 3 | FM-04, FM-12, FM-13 | Mandatory |

*TSR-AIQL-009, -010, -011 are cross-cutting requirements (WCET, process, diagnostics) that support the overall safety mechanism integrity rather than addressing specific failure modes.

**Coverage check**: All 20 TSRs (15 mandatory + 5 optional) address at least 1 failure mode or serve a cross-cutting safety purpose. **PASS**

### 13.4 Verification Coverage

| Verification Item | Failure Modes Covered | TSRs Verified | Status |
|-------------------|----------------------|---------------|--------|
| FI-01 | FM-01, FM-03 | TSR-AIQL-001, -002 | Mandatory |
| FI-02 | FM-03 | TSR-AIQL-001 | Mandatory |
| FI-03 | FM-02 | TSR-AIQL-003 | Mandatory |
| FI-04 | FM-07 | TSR-AIQL-004 | Mandatory |
| FI-05 | FM-04 | TSR-AIQL-005 | Mandatory |
| FI-07 | All (data integrity) | TSR-AIQL-007, -008 | Mandatory |
| FI-08 | FM-08 | TSR-AIQL-012 | Mandatory |
| FI-09 | FM-08 | TSR-AIQL-013 | Mandatory |
| FI-10 | FM-09 | TSR-AIQL-015, -019 | Optional |
| FI-11 | FM-10 | TSR-AIQL-016 | Optional |
| FI-12 | FM-09, FM-10 | TSR-AIQL-017 | Optional |
| FI-13 | FM-10, FM-11 | TSR-AIQL-018 | Optional |
| FI-14 | FM-09, FM-10 | TSR-AIQL-019 | Optional |
| FI-15 | FM-01, FM-04, FM-12 | TSR-AIQL-020, -022 | Mandatory |
| FI-16 | FM-04, FM-12, FM-13 | TSR-AIQL-021 | Mandatory |
| FI-17 | FM-12 | TSR-AIQL-020, -021, -022 | Mandatory |

**Coverage check**: All data integrity failure modes (FM-01 through FM-04, FM-07, FM-08), passive diagnostic failure modes (FM-12, FM-13), and optional BIST failure modes (FM-09, FM-10, FM-11) covered by at least one fault injection test. FM-05/FM-06 verified at system level. **PASS**

---

## 14. Open Items

| # | Open Item | Owner | Priority | Target Date | Status |
|---|-----------|-------|----------|-------------|--------|
| OI-01 | **Audio frame interface specification**: Finalize the exact audio frame format, field ordering, and byte alignment with the audio driver development team. Current Section 4.2 is a draft proposal. | Audio SW Lead + Safety Engineering | High | TBD | Open |
| OI-02 | **HARA alignment for SG-EMV-01**: ~~HARA-0003 row added (S2/E3/C2 → ASIL B)~~ TSC-001 now aligned to Audio_HARA.xlsx (UC-React-to-EMV), row: "Yield to Activated EMV (Highway), siren+lights, slow/stationary in front" — S3/E2/C3 → ASIL B (v0.7). Remaining: formal HARA review board approval to baseline safety goal SG-EMV-01; confirm HARA ID assignment convention. | Safety Manager | High | TBD | Partially Resolved |
| OI-03 | **Target compute platform**: Confirm DRIVE AGX Thor (Blackwell) as the target platform for AIQL deployment within Hyperion 10 reference architecture. WCET (TSR-AIQL-009) and MPU/MIG configuration (TSR-AIQL-012) are platform-dependent. Confirm MIG partition allocation for AIQL safety-critical workload. | Platform Architecture | High | TBD | Open |
| OI-04 | **Microphone array specifications**: ~~Confirm microphone placement (exterior vs. cabin)~~ ~~Microphone placement confirmed as in-cabin (v0.4).~~ Microphone placement updated to **exterior** (v0.9) per Hyperion 10 reference architecture. Remaining: finalize count (minimum 2, recommended 4+), model selection, IP67 environmental qualification, and acoustic specifications for exterior placement. AoU-009 updated for exterior requirements. See OI-17 for program-specific mic count and placement confirmation. | Audio HW Lead | Medium | TBD | Partially Resolved |
| OI-05 | **Siren classifier output format**: Confirm the classifier output structure (Section 4.2.3) with the audio ML team. The direction-of-arrival field and confidence metric format need alignment. | Audio ML Team | Medium | TBD | Open |
| OI-06 | **FTTI confirmation**: Validate the 500 ms FTTI through system-level timing analysis including worst-case audio processing latency, AMO inference time, and vehicle dynamic response. Current allocation (Section 10.4) is preliminary. | System Safety + Vehicle Dynamics | High | TBD | Open |
| OI-07 | **System-level verification of FM-05/FM-06**: Verify that AMO multi-sensor fusion adequately mitigates classifier false negatives (FM-05) and false positives (FM-06) when audio is used as a secondary sensor. This is outside the AIQL scope but must be confirmed at the system level. | System Safety + Perception Team | High | TBD | Open |
| OI-08 | **Speaker specifications for BIST** *(customer-dependent — only applicable if exterior speaker is available for acoustic loopback)*: Select specific exterior speaker for BIST use. Confirm frequency response, SPL output at mic positions, THD, and independent DAC access (AoU-011). Determine if existing exterior speaker (backup warning, AVAS) can be shared or if a dedicated BIST speaker is needed. Not applicable for Hyperion reference platform (no speaker). | Audio HW Lead | Low (customer-dependent) | TBD | Open — customer-dependent |
| OI-09 | **Acoustic characterization** *(customer-dependent for BIST; mandatory for passive checks)*: For BIST-equipped platforms: perform acoustic transfer function measurement from BIST speaker to each exterior microphone across 500-3000 Hz; characterize variation across temperature, wind speed, and ambient noise conditions. For all platforms: characterize exterior microphone noise profiles for noise profile monitoring (TSR-AIQL-020) calibration — measure expected noise floor, spectral shape, and cross-mic RMS variation at representative vehicle speeds. | Audio HW Lead + Safety Engineering | Medium | TBD | Open |
| OI-10 | **BIST signal cancellation validation** *(customer-dependent — only applicable if acoustic loopback BIST is enabled)*: Validate that the BIST signal cancellation algorithm (TSR-AIQL-018) achieves >= 30 dB attenuation in representative environmental conditions. Determine if adaptive cancellation is needed or if a fixed reference subtraction is sufficient. Not applicable for Hyperion reference platform. | Safety Engineering + Audio SW Lead | Low (customer-dependent) | TBD | Open — customer-dependent |
| OI-11 | **BIST false failure rate target** *(customer-dependent — only applicable if acoustic loopback BIST is enabled)*: Define quantitative target for false BIST failure rate (preliminary target: < 1 per 1000 driving hours). Perform statistical analysis of expected false failure rate based on environmental noise profiles and BIST signal design. If target is not met, consider noise-adaptive BIST scheduling (OI-14). Not applicable for Hyperion reference platform. | Safety Engineering | Low (customer-dependent) | TBD | Open — customer-dependent |
| OI-12 | **MRM interface specification**: Define the exact interface between AIQL and THOR FSI for MRM requests via the SOC_Error / FSI_I2C safety signaling path. Specify signal format, error codes, latency requirements, and acknowledgment protocol. See also OI-16 for THOR FSI path confirmation. | Safety Engineering + Vehicle Platform | High | TBD | Open |
| OI-13 | **Periodic BIST interval optimization** *(customer-dependent — only applicable if acoustic loopback BIST is enabled)*: The 60-second periodic BIST interval is a preliminary value. Optimize based on: (a) expected failure rate dynamics (how fast can acoustic coupling degrade?); (b) environmental audio intrusion tolerance; (c) BIST signal cancellation effectiveness; (d) false failure rate impact. Consider adaptive interval based on vehicle state (longer at highway, shorter in city). Not applicable for Hyperion reference platform. | Safety Engineering | Low (customer-dependent) | TBD | Open — customer-dependent |
| OI-14 | **ANC interaction with BIST** *(customer-dependent — only applicable if acoustic loopback BIST is enabled and vehicle has ANC)*: If the vehicle has Active Noise Cancellation (ANC), determine whether ANC will attenuate or distort the BIST signal. If so, define coordination protocol (suspend ANC during BIST window, or account for ANC transfer function in spectral match reference). Not applicable for Hyperion reference platform. | Audio HW Lead + Safety Engineering | Low (customer-dependent) | TBD | Open — customer-dependent |
| OI-15 | **A2B frame structure confirmation**: Confirm CRC, alive counter, sequence counter, and timestamp fields are present in A2B frames as delivered to Thor Main (per AoU-013). Confirm with systems engineering that the A2B transceiver (Maarva) preserves these fields without modification. Verify end-to-end integrity across A2B bus. | Systems Engineering + Audio HW Lead | High | TBD | Open |
| OI-16 | **THOR FSI safety signaling path**: Confirm SOC_Error / FSI_I2C interface specification for AIQL error reporting and MRM requests to THOR FSI. Define error codes for AIQL qualification failures (passive check failure, BIST failure if applicable). Confirm latency from SOC_Error assertion to THOR FSI recognition. | Safety Engineering + Platform Architecture | High | TBD | Open |
| OI-17 | **Exterior mic quantity and placement**: Confirm number and placement of exterior microphones for upcoming programs. Minimum 2 required for cross-mic plausibility (TSR-AIQL-021); recommended 4+ for robust coverage and single-channel fault tolerance. Placement affects wind noise exposure, debris risk, and siren detection coverage. Coordinate with vehicle integration and acoustic design teams. | Audio HW Lead + Vehicle Integration | High | TBD | Open |
| OI-18 | **NDAS EVD implementation commitment**: NDAS perception team has not committed to Emergency Vehicle Detection (EVD) as a shipping feature. The AIQL qualifies audio input for AMO, but AMO's EVD module (siren fusion with primary sensors) must be implemented for the audio input to contribute to the safety goal SG-EMV-01. Resolve ownership of EVD implementation before finalizing this TSC. If EVD is not implemented, the AIQL still provides FFI for the audio input path but the end-to-end safety argument for SG-EMV-01 is incomplete. | Safety Engineering + NDAS Perception Team | Critical | TBD | Open |

---

## 15. Appendices

### Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **AIQL** | Audio Input Qualification Layer — the ASIL B safety mechanism defined in this TSC |
| **AMO (Alpamayo)** | NVIDIA's on-vehicle perception system running on DRIVE AGX Thor. AMO is the direct consumer of microphone input qualified by the AIQL. AMO fuses audio with camera, LiDAR, radar, and ultrasonic data for autonomous driving perception. ASIL B on the AIQL derives from FFI requirements within AMO. Alpamayo also refers to the cloud-based foundation-model research layer (10B-parameter VLA model trained on DGX) that produces distilled compact models deployed into AMO on-vehicle. |
| **DRIVE AGX Thor** | NVIDIA's centralized AV compute platform based on the Blackwell architecture (2x Thor SoCs, 2000+ FP4 TFLOPS). Hosts DRIVE OS, DriveWorks, and NDAS DRIVE AV. |
| **DRIVE Hyperion 10** | NVIDIA's production-ready AV reference architecture: DRIVE AGX Thor compute + qualified sensor suite (14 cameras, 9 radars, 1 lidar, 12 ultrasonics, 1 exterior mic array, 4 interior cameras). |
| **NDAS (DRIVE AV)** | NVIDIA DRIVE AV Solution — the full-stack autonomous driving software (prediction, mapping, planning, control, Safety Force Field) running on DRIVE AGX Thor. AMO feeds perception outputs into NDAS for driving decisions. |
| **A2B** | Automotive Audio Bus — a high-bandwidth, low-latency digital audio bus used to connect microphones, speakers, and other audio peripherals to the SoC. Supports up to 16 channels on the Maarva transceiver. |
| **ANC** | Active Noise Cancellation — system that generates anti-phase sound to reduce cabin noise |
| **AoU** | Assumption of Use — interface contract on the QM audio subsystem per ISO 26262-8 Clause 12 |
| **ASIL** | Automotive Safety Integrity Level — risk classification per ISO 26262 (QM, A, B, C, D) |
| **BIST** | Built-In Self-Test — a self-diagnostic mechanism that verifies hardware path integrity using a known test signal. In this architecture, acoustic loopback BIST is **optional and customer-dependent** — it requires an exterior speaker. The primary diagnostic approach uses passive qualification checks (noise profile, cross-mic plausibility, input sanitization). |
| **BP** | Base Platform — Thor Main software partition. The AIQL runs on Thor Main as part of the BP SW. |
| **CRC-32** | Cyclic Redundancy Check with 32-bit polynomial — used for data integrity verification |
| **dBFS** | Decibels relative to Full Scale — amplitude measurement where 0 dBFS is the maximum digital level |
| **E2E** | End-to-End protection — communication protection mechanism per AUTOSAR |
| **FFI** | Freedom From Interference — ensures faults in one element cannot corrupt another; analyzed in spatial, temporal, and communication dimensions |
| **FIT** | Failures In Time — failure rate unit (1 FIT = 1 failure per 10^9 hours) |
| **FM** | Failure Mode — a specific way the audio input path can fail |
| **FTTI** | Fault Tolerant Time Interval — maximum time from fault occurrence to safe state |
| **HiL** | Hardware-in-the-Loop — test environment with real target hardware and simulated vehicle |
| **LFM** | Latent Fault Metric — percentage of latent faults covered by safety mechanisms |
| **Maarva** | ECU / A2B transceiver that accepts A2B microphone input (up to 16 channels) and delivers audio frames to Thor Main SoC. Part of the Base Platform board. |
| **MC/DC** | Modified Condition/Decision Coverage — structural coverage metric required for ASIL B |
| **MMU** | Memory Management Unit — hardware mechanism for virtual memory and protection |
| **MPU** | Memory Protection Unit — hardware mechanism for spatial memory isolation |
| **MRM** | Minimal Risk Maneuver — a controlled vehicle stop initiated when a safety-critical fault cannot be recovered |
| **PCM** | Pulse-Code Modulation — standard digital audio encoding format |
| **QM** | Quality Management — lowest integrity level per ISO 26262 (no specific safety requirements) |
| **SEooC** | Safety Element out of Context — element developed without complete safety context; requires AoUs for integration |
| **SG** | Safety Goal — top-level safety requirement derived from HARA |
| **SiL** | Software-in-the-Loop — test environment with simulated hardware |
| **Sine Sweep** | A test signal whose frequency increases continuously over time, used to characterize frequency response |
| **SNR** | Signal-to-Noise Ratio — measure of signal quality in decibels |
| **SoC** | System on Chip — integrated circuit containing multiple processing elements |
| **SOTIF** | Safety of the Intended Functionality — per ISO 21448, addresses performance limitations rather than faults |
| **SPFM** | Single Point Fault Metric — percentage of single-point faults covered by safety mechanisms |
| **TC** | Triggering Condition — SOTIF concept for conditions that trigger performance limitations |
| **THD** | Total Harmonic Distortion — ratio of harmonic content to fundamental, measuring signal purity |
| **THOR FSI** | Thor Functional Safety Island — the safety MCU on the Base Platform board. Receives safety signaling from Thor Main (BP SW) via SOC_Error / FSI_I2C interfaces. Responsible for watchdog monitoring, MRM request forwarding, and system-level safety management. |
| **TSC** | Technical Safety Concept — system-level safety design document per ISO 26262-4 |
| **TSR** | Technical Safety Requirement — individual requirement within a TSC |
| **WCET** | Worst-Case Execution Time — maximum time a software function takes to execute |

### Appendix B: Audio Frame Format (Binary Layout)

```
Offset  Size    Field                   Type        Notes
------  ------  ----------------------  ----------  --------------------------------
0x00    4       frame_id                uint32_le   Monotonically increasing
0x04    8       timestamp_us            uint64_le   Microseconds, system monotonic
0x0C    1       alive_counter           uint8       Modulo 256
0x0D    2       sequence_counter        uint16_le   Wraps at 65535
0x0F    4       sample_rate_hz          uint32_le   Expected: 48000
0x13    1       num_channels            uint8       Expected: 4
0x14    2       num_samples             uint16_le   Samples per channel
0x16    N*2     audio_data              int16_le[]  Interleaved PCM; N = num_channels * num_samples
0x16+N*2  4     siren_probability       float32_le  [0.0, 1.0]
...+4   4       horn_probability        float32_le  [0.0, 1.0]
...+8   4       direction_deg           float32_le  [-180.0, +180.0]
...+12  4       confidence              float32_le  [0.0, 1.0]
...+16  4       crc32                   uint32_le   CRC-32 over bytes [0x00, ...+15]

Total fixed header: 22 bytes
Total classifier: 16 bytes
Total CRC: 4 bytes
Total frame: 42 + (num_channels * num_samples * 2) bytes

Example at 48 kHz, 4 channels, 2400 samples/channel (50 ms @ 48 kHz):
  Audio data size: 4 * 2400 * 2 = 19200 bytes
  Total frame size: 42 + 19200 = 19242 bytes
  At 20 Hz: 19242 * 20 = 384840 bytes/sec (~375 KB/s)
```

**Note**: The BIST output interface (Section 4.2.5) uses a separate output path to the codec DAC and is not included in the audio frame format above. BIST samples are generated internally by the AIQL and transmitted directly to the speaker via the codec DAC interface — they do not pass through the QM audio driver.

### Appendix C: ASIL Decomposition Evidence

The following table provides the evidence chain supporting the ASIL decomposition argument in Section 8.

| Evidence Item | ISO 26262 Clause | Content | Status |
|--------------|-------------------|---------|--------|
| Decomposition scheme | Part 9, 5.4.2 | ASIL B = ASIL B(AIQL on Thor Main BP) + QM(Audio HW) + QM(Audio SW) + QM(Speaker HW, optional) — documented in Section 8.1 | Complete |
| Independence argument | Part 9, 5.4.3 | Hardware, software, and common cause analysis — documented in Section 8.2 | Complete |
| Sufficient safety mechanisms | Part 9, 5.4.4 | 20 TSRs (15 mandatory ASIL B + 5 optional ASIL B + 1 QM) covering all data integrity, passive diagnostic, and BIST failure modes — documented in Section 7; FM-05/FM-06 addressed at system level | Complete |
| Assumptions of Use | Part 8, 12.4.2 | 14 AoUs (12 mandatory + 2 optional) on QM audio subsystem — documented in Section 9 | Complete |
| FFI analysis | Part 9, 7 | Spatial, temporal, communication FFI — documented in Section 6.2 | Complete |
| Dependent failure analysis | Part 9, 7 | Common cause analysis — documented in Section 8.2.3 | Complete |
| Verification of decomposition | Part 9, 5.4.5 | Fault injection campaign (123 mandatory + 50 optional = 173 total tests) — documented in Section 11.3 | Planned |

### Appendix D: E2E Protection Profile Rationale

The AIQL uses an end-to-end protection profile similar to AUTOSAR E2E Profile 1, adapted for the audio frame interface. The rationale for each E2E mechanism is as follows:

| E2E Mechanism | AUTOSAR E2E Equivalent | Purpose | Failure Modes Addressed |
|---------------|----------------------|---------|------------------------|
| CRC-32 | E2E Profile 1 CRC | Detect data corruption in transit from QM driver to AIQL | FM-02 (Corruption) |
| Alive counter (8-bit) | E2E Profile 1 Counter | Detect loss of communication (driver crash, bus failure) | FM-01 (Loss) |
| Sequence counter (16-bit) | — (extended beyond E2E Profile 1) | Detect reordering, duplication, and replay of frames | FM-07 (Sequence Disorder) |
| Timestamp freshness | — (not in standard E2E Profile 1) | Detect stale data from buffer stalls or scheduling issues | FM-03 (Staleness) |

**Why not standard AUTOSAR E2E Profile 1?**

Standard E2E Profile 1 is designed for CAN/FlexRay messages (8-64 bytes). The audio frame interface has different characteristics:

1. **Large payload**: Audio frames are ~19 KB vs. 8-64 bytes for CAN messages — CRC-32 is appropriate but the alive counter and sequence counter need different semantics
2. **Shared memory transport**: Audio frames are exchanged via shared memory, not CAN/FlexRay — no bus-level error detection to supplement E2E
3. **High data rate**: 20 Hz with 19 KB frames (~375 KB/s) — requires efficient checking within 5 ms WCET
4. **Timestamp required**: Audio freshness is critical for siren detection temporal correlation — timestamps are essential but not part of standard E2E Profile 1

### Appendix E: BIST Loopback Architecture (Optional, Customer-Dependent)

**Note**: This appendix describes the acoustic loopback BIST architecture that is only active when an exterior speaker (e.g., backup warning speaker, AVAS/pedestrian alerting speaker) is available on the vehicle platform. The Hyperion reference platform does not include an exterior speaker for BIST. If no speaker is present, BIST is disabled and passive qualification checks (TSR-AIQL-020, -021, -022) serve as the sole diagnostic approach.

#### E.1 BIST Signal Path

```
+==================+     +============+     +===============+     +=============+
| BIST Signal      |---->| Codec DAC  |---->| Exterior      |---->| Acoustic    |
| Generator        |     |            |     | Speaker       |     | Air Path    |
| (within AIQL)    |     | (QM HW)    |     | (QM HW)      |     | (exterior)  |
+==================+     +============+     +===============+     +=============+
                                                                       |
                              +----------------------------------------+
                              |
                              v
+=============+     +============+     +==================+
| Exterior    |---->| Codec ADC  |---->| BIST Spectral    |
| Microphone  |     | (via A2B)  |     | Match Analyzer   |
| (QM HW)     |     | (QM HW)    |     | (within AIQL)    |
+=============+     +============+     +==================+
                                              |
                                              v
                                       [PASS / FAIL]
```

#### E.2 BIST Timing Diagram

```
Startup Sequence:
  t=0       System power-on
  t=0+      AIQL initializes in NOT_QUALIFIED state
  t=Tinit   Startup BIST begins (Tinit = system init time)
            |<---------- 2000 ms sweep --------->|<-50ms->|
            [Sine sweep 500-3000 Hz at -20 dBFS] [Analyze]
            |                                             |
            +--- PASS ---> Begin passive checks ---> QUALIFIED
            |
            +--- FAIL ---> Retry (up to 3 times)
                           |
                           +--- All retries fail ---> MRM_REQUESTED

Periodic Sequence (during QUALIFIED operation):
  t=0       Last BIST completed
  t=60s     Periodic BIST triggers
            |<-- 200 ms burst -->|<-50ms->|
            [Abbreviated sweep ] [Analyze]
            |                            |
            +--- PASS ---> Continue QUALIFIED
            |
            +--- FAIL ---> Retry (up to 2 times)
                           |
                           +--- All retries fail ---> NOT_QUALIFIED ---> MRM_REQUESTED
```

#### E.3 Spectral Match Algorithm

The BIST spectral match verification performs two independent checks:

1. **Normalized Cross-Correlation**:
   - Compute FFT of received microphone signal during BIST window
   - Compute expected spectral envelope based on known BIST signal + calibrated cabin transfer function
   - Calculate normalized cross-correlation coefficient between expected and received spectral envelopes
   - Threshold: correlation >= 0.85

2. **1/3-Octave Band Amplitude Check**:
   - Divide 500-3000 Hz range into 1/3-octave bands (approximately 8 bands)
   - Compute received energy per band
   - Compare to expected energy per band (from calibration)
   - Threshold: deviation within +/- 6 dB per band

Both checks must pass for BIST to pass. This dual-check approach detects both broadband degradation (correlation drop) and frequency-selective faults (band deviation).

#### E.4 BIST State Flowchart

```
                    +-------------------+
                    |   AIQL Power On   |
                    +--------+----------+
                             |
                             v
                    +-------------------+
                    | NOT_QUALIFIED     |
                    | (Initial State)   |
                    +--------+----------+
                             |
                             v
                    +-------------------+
                    | Execute Startup   |
                    | BIST (2s sweep)   |
                    +--------+----------+
                             |
                    +--------+----------+
                    |                   |
                    v                   v
              [BIST PASS]        [BIST FAIL]
                    |                   |
                    v                   v
              Begin passive      +------------------+
              checks             | Retry counter    |
                    |            | < 3?             |
                    v            +--------+---------+
              +============+             |         |
              | QUALIFIED  |           Yes        No
              +============+             |         |
                    |                    v         v
                    | Every 60s    Retry BIST  MRM_REQUESTED
                    v                             (Terminal)
              +-------------------+
              | Execute Periodic  |
              | BIST (200ms burst)|
              +--------+----------+
                       |
              +--------+----------+
              |                   |
              v                   v
        [BIST PASS]        [BIST FAIL]
              |                   |
              v                   v
        Continue            +------------------+
        QUALIFIED           | Retry counter    |
                            | < 2?             |
                            +--------+---------+
                                     |         |
                                   Yes        No
                                     |         |
                                     v         v
                               Retry BIST  NOT_QUALIFIED
                                           --> MRM_REQUESTED
                                               (Terminal)
```

#### E.5 Configurable BIST Parameters

| Parameter | Default Value | Range | Rationale |
|-----------|--------------|-------|-----------|
| Startup sweep duration | 2000 ms | 500-5000 ms | Longer sweep = better frequency resolution; shorter = faster startup |
| Periodic burst duration | 200 ms | 100-1000 ms | Shorter = less cabin intrusion; longer = better spectral resolution |
| Periodic interval | 60 s | 10-300 s | Shorter = better coverage; longer = less intrusion |
| Sweep start frequency | 500 Hz | 200-1000 Hz | Must cover siren frequency range lower bound |
| Sweep end frequency | 3000 Hz | 2000-5000 Hz | Must cover siren frequency range upper bound |
| Output amplitude | -20 dBFS | -30 to -10 dBFS | Lower = less audible; higher = better SNR |
| Correlation threshold | 0.85 | 0.70-0.95 | Lower = fewer false failures; higher = better fault detection |
| Amplitude tolerance | +/- 6 dB | +/- 3 to +/- 12 dB | Tighter = better fault detection; looser = fewer false failures |
| Startup retries | 3 | 1-5 | More retries = fewer false MRMs; fewer = faster failure detection |
| Periodic retries | 2 | 1-3 | More retries = fewer false MRMs; fewer = faster failure detection |
| Signal cancellation target | 30 dB | 20-40 dB | Higher = less BIST leakage to AMO; harder to achieve |
