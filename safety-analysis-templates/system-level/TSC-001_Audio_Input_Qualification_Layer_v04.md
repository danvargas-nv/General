# TSC-001: Audio Input Qualification Layer (AIQL)

## 1. Header & Metadata

| Field | Value |
|-------|-------|
| **Document ID** | TSC-001 |
| **Title** | Audio Input Qualification Layer (AIQL) — Technical Safety Concept |
| **ASIL Target** | ASIL B |
| **Safety Goal** | SG-03 |
| **HARA Reference** | HARA-0003 |
| **Item** | NDAS DRIVE AV Stack — Audio Input Path (on DRIVE AGX Thor / Hyperion 10) |
| **Version** | 0.5 |
| **Status** | In Development |
| **Author** | Safety Engineering |
| **Created** | 2026-02-22 |
| **Last Modified** | 2026-02-23 |
| **Classification** | Confidential |

### Document Traceability

| Relationship | Document |
|-------------|----------|
| Parent | HARA-001 — Hazard Analysis and Risk Assessment |
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

---

## 2. Scope & Purpose

### 2.1 Scope

This Technical Safety Concept (TSC) defines the **Audio Input Qualification Layer (AIQL)** — an ASIL B software component at the boundary between the QM-rated audio subsystem and the ASIL-rated NDAS DRIVE AV perception pipeline, running on the DRIVE AGX Thor compute platform within the DRIVE Hyperion 10 reference architecture.

**Platform context**: Alpamayo is NVIDIA's foundation-model AV research layer that sits above NDAS/DRIVE AV. Alpamayo trains in the cloud (on DGX infrastructure) and produces distilled compact models that deploy into the NDAS DRIVE AV on-vehicle stack. The AIQL's downstream consumer is the NDAS DRIVE AV perception pipeline — not Alpamayo directly. Alpamayo-distilled models may run within that pipeline, but the system boundary for this TSC is the NDAS DRIVE AV interface.

Audio is designated as a **secondary safety sensor**. The primary sensors for emergency vehicle detection and all driving decisions are camera, LiDAR, and radar. Audio provides supplementary siren/horn detection that enhances overall detection confidence but is not the sole or primary input for any safety-critical decision. NDAS DRIVE AV multi-sensor fusion architecture is responsible for combining all sensor inputs and determining the appropriate driving response.

The AIQL qualifies QM audio input at the I/O boundary by implementing safety mechanisms that detect, contain, and mitigate data integrity failure modes in the audio input path. This follows the element-out-of-context (SEooC) integration approach per ISO 26262-8 Clause 12.

#### In Scope

- Qualification of the audio data stream at the NDAS DRIVE AV input boundary
- Freshness, integrity, sequence, and range validation of audio frames
- Safe state definition and graceful degradation strategy
- Temporal and spatial Freedom From Interference (FFI) mechanisms
- Fault injection verification campaign for the AIQL
- ASIL decomposition argument per ISO 26262-9
- Speaker-Microphone Loopback Built-In Self-Test (BIST) for audio path integrity verification

#### Out of Scope

- Design or modification of the QM audio hardware (microphones, codec, amplifiers)
- Design or modification of the QM audio driver or DSP firmware
- The siren/horn classifier algorithm itself (treated as QM element with AoUs)
- Classifier correctness validation (semantic accuracy of siren/horn detection is a system-level concern addressed by NDAS DRIVE AV multi-sensor fusion with primary sensors)
- Cross-modal plausibility checking with camera or other sensors (sensor fusion is NDAS DRIVE AV's responsibility)
- NDAS DRIVE AV internals beyond the audio input interface
- Alpamayo foundation model training, distillation, or cloud infrastructure
- Non-safety audio functions (in-cabin entertainment, voice commands)
- Cybersecurity requirements for the audio path (addressed in separate cybersecurity concept)

### 2.2 Purpose

The NDAS DRIVE AV perception pipeline — running on DRIVE AGX Thor within the Hyperion 10 reference architecture — receives audio input for siren/horn detection that influences safety-critical driving decisions, specifically yielding to emergency vehicles. The perception pipeline may incorporate Alpamayo-distilled models, but from this TSC's perspective the system boundary is the NDAS DRIVE AV interface. The audio hardware and software (microphone, codec, DSP, driver, classifier) are developed to QM (Quality Management) level.

Without qualification, unqualified audio input creates Freedom From Interference (FFI) issues within the NDAS DRIVE AV stack. Specifically:

1. **Corrupted audio data** could inject erroneous siren/horn signals into NDAS DRIVE AV's fusion layer, degrading fusion confidence
2. **Missing or stale audio data** could cause NDAS DRIVE AV to operate with outdated supplementary information without awareness of its invalidity

The AIQL resolves these FFI issues by implementing ASIL B safety mechanisms at the I/O boundary, qualifying the QM audio input before it enters the NDAS DRIVE AV perception pipeline. The AIQL ensures that audio data entering NDAS DRIVE AV is fresh, intact, and within expected ranges — or explicitly marked as not qualified so the fusion layer can exclude it. This avoids the prohibitively expensive alternative of developing the entire audio hardware and software stack to ASIL B.

Additionally, the AIQL incorporates a **Speaker-Microphone Loopback Built-In Self-Test (BIST)** — a pre-processing integrity verification mechanism. The BIST uses in-cabin speakers to play a known sine sweep signal that is picked up by the in-cabin microphones, allowing the AIQL to validate the complete acoustic-to-digital audio path before and during operation. On confirmed BIST failure, the vehicle initiates a Minimal Risk Maneuver (MRM) — a controlled stop — because BIST failure indicates a fundamental loss of audio path integrity that cannot be recovered through software-only mechanisms.

**Note**: Classifier correctness (whether the siren/horn classifier accurately identifies emergency vehicle audio) is a system-level concern. Because audio is a secondary sensor, misclassification is mitigated by NDAS DRIVE AV multi-sensor fusion with primary sensors (camera, LiDAR, radar), not by the AIQL.

### 2.3 Boundary Definition

```
                    QM Boundary                          ASIL B Boundary
                    |                                    |
  [In-Cabin   ] --> [Codec] --> [Audio Driver] --> [AIQL] --> [NDAS DRIVE AV]
  [Microphones]     |(ADC+DSP)|   (QM SW)          (ASILB)   (ASIL, on Thor)
                    |                                    |
     QM HW          QM SW        QM SW          ASIL B SW    ASIL (NDAS)

  [In-Cabin   ] <-- [Codec] <-- [BIST Signal  ] <-+
  [Speakers   ]     |(DAC)  |   | Generator    |   |
                    |           | (within AIQL)|   |
     QM HW          QM HW       ASIL B SW         |
                                                   |
                    Loopback: Speaker output -------> Microphone input
                              (verified by AIQL spectral match)
```

The AIQL sits at the exact boundary between the QM audio subsystem and the ASIL-rated NDAS DRIVE AV perception pipeline. All audio data entering NDAS DRIVE AV passes through the AIQL — there is no bypass path.

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
+==============+     +============+     +=============+     +======+     +==========+
|  In-Cabin    |---->|   Codec    |---->| Audio Driver|---->| AIQL |---->| NDAS     |
|  Microphone  |     | (ADC+DSP)  |     |  (QM SW)    |     |(ASILB)|    | DRIVE AV |
|  Array       |     +============+     +=============+     +======+     +==========+
+==============+          |                   |               |   |           |
    ^ |                   |                   |               |   |           |
    | | Analog audio      | Digital PCM       | Audio frame   |   | Qualified |
    | | (electrical)      | (I2S/TDM)         | (shared mem)  |   | audio     |
    | |                   |                   |               |   | frame +   |
    | |                   |                   |               |   | status    |
    | |                   |                   |               |   |           |
    | +--- QM HW ---------+---- QM SW --------+               |   |           |
    |                                                          |   |           |
    |   BIST Loopback Path (speaker -> acoustic -> mic)        |   |           |
    |                                                          |   |           |
    +<-- [Codec (DAC)] <-- [In-Cabin Speakers] <---------------+   |           |
         QM HW              QM HW            BIST Signal Gen       |           |
                                             (within AIQL)         |           |
                                                                   |           |
              +====================================================+           |
              | AIQL Internal Architecture                         |           |
              |                                                    |           |
              | +-------------+  +---------------+  +------------+ |           |
              | | Freshness & |  | Integrity &   |  | BIST       | |           |
              | | Alive Check |  | Range Check   |  | Loopback   | |           |
              | |             |  |               |  | Module     | |           |
              | | - Timestamp |  | - CRC-32      |  |            | |           |
              | | - Alive Ctr |  | - Seq Counter |  | - Signal   | |           |
              | | - Freshness |  | - Range Valid |  |   Generator| |           |
              | +------+------+  +-------+-------+  | - Spectral | |           |
              |        |                 |           |   Match    | |           |
              |        |                 |           | - Pass/Fail| |           |
              |        |                 |           +------+-----+ |           |
              |        v                 v                  v       |           |
              | +---------------------------------------------------+          |
              | | Qualification Decision & State Machine              |          |
              | |                                                    |          |
              | | QUALIFIED ---> DEGRADED ---> NOT_QUALIFIED         |          |
              | |                                 |                  |          |
              | |                                 +--> MRM_REQUESTED |          |
              | +---------------------------------------------------+          |
              |                                                    |           |
              +====================================================+           |
                                                                   |           |
                      +--------------------------------------------+           |
                      | Output: qualified_audio_frame + qualification_status   |
                      +--------------------------------------------------------+
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

#### 4.2.2 Output Interface: AIQL → NDAS DRIVE AV

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
| 7 | `FF_BIST_FAIL` | BIST loopback verification failed |
| 8 | `FF_BIST_SPEAKER` | BIST speaker subsystem failure (no output) |
| 9 | `FF_BIST_COUPLING` | BIST acoustic coupling degradation |
| 10-15 | Reserved | Reserved for future use |

#### 4.2.5 BIST Output Interface: AIQL → Speaker (via Codec DAC)

| Field | Type | Size (bytes) | Description |
|-------|------|-------------|-------------|
| `bist_sequence_id` | uint16 | 2 | Unique BIST test sequence identifier |
| `signal_type` | enum8 | 1 | SINE_SWEEP (0x01), SINGLE_TONE (0x02) |
| `start_freq_hz` | uint16 | 2 | Sweep start frequency (default: 500 Hz) |
| `end_freq_hz` | uint16 | 2 | Sweep end frequency (default: 3000 Hz) |
| `amplitude_dbfs` | int8 | 1 | Output amplitude in dBFS (default: -20) |
| `duration_ms` | uint16 | 2 | Signal duration in milliseconds |
| `bist_samples` | int16[] | variable | PCM samples for speaker output |

### 4.3 Timing Architecture

```
Audio Frame Period: 50 ms (20 Hz frame rate)

|<--- 50 ms frame period --->|
|                             |
[Audio Capture] [Processing] [AIQL Qualification] [NDAS DRIVE AV Ingestion]
|<-- 20 ms --->|<-- 10 ms ->|<----- 5 ms ------>|<----- 5 ms ------->|
                                                                       |
                              10 ms margin for jitter and scheduling --+

FTTI Budget Allocation (500 ms total):
  - Fault detection by AIQL:          <= 100 ms (2 audio frames)
  - State transition to DEGRADED:     <=  50 ms (1 frame)
  - NDAS DRIVE AV reaction (replan):  <= 150 ms
  - Vehicle dynamic response:         <= 200 ms
  Total:                                  500 ms

BIST Timing:
  Startup BIST:
    - Full sine sweep 500-3000 Hz:    2000 ms duration
    - Spectral match analysis:          50 ms
    - Total startup BIST:             2050 ms (must pass before QUALIFIED)
    - Retries on failure:             up to 3 (total max: ~8.2 s)

  Periodic BIST:
    - Burst signal (abbreviated sweep): 200 ms duration
    - Interval:                          every 60 seconds
    - Spectral match analysis:            50 ms
    - Signal cancellation for NDAS DRIVE AV: within same 200 ms window
```

---

## 5. Safety Goal & ASIL Determination

### 5.1 Safety Goal Definition

**SG-03: The audio input path shall provide qualified siren/horn detection data to the NDAS DRIVE AV perception pipeline such that the autonomous vehicle can detect and yield to emergency vehicles within the operational design domain.**

| Attribute | Value |
|-----------|-------|
| Safety Goal ID | SG-03 |
| Item | Audio Input Path (Microphone → Codec → Driver → AIQL → NDAS DRIVE AV) |
| Hazardous Event | Failure to yield to approaching emergency vehicle due to undetected or misclassified siren/horn audio |
| Sensor Role | **Secondary** — audio supplements primary sensors (camera, LiDAR, radar) for emergency vehicle detection. NDAS DRIVE AV multi-sensor fusion uses audio as an additional input channel; driving decisions are not solely dependent on audio. |
| HARA Reference | HARA-0003 |

### 5.2 ASIL Determination

The ASIL for SG-03 is determined per ISO 26262-3 Table 4 using the following parameters:

#### Severity: S2 (Severe and life-threatening injuries, survival probable)

**Justification**: Failure to yield to an emergency vehicle at urban speeds (30-60 km/h) can result in a collision between the ego vehicle and the emergency vehicle, or force the emergency vehicle into an evasive maneuver that impacts other road users. The collision geometry (typically T-bone or side-impact at intersections) at urban speeds results in severe but typically survivable injuries. S3 is not assigned because:
- Ego vehicle speeds in yielding scenarios are typically moderate (urban environment)
- Emergency vehicles are braking/maneuvering, reducing closing speeds
- The scenario involves failure to yield, not failure to brake for a direct collision

#### Exposure: E3 (Medium probability — occurs in most driving scenarios)

**Justification**: Emergency vehicle encounters are a common driving event. Urban driving constitutes a significant portion of operational time, and encounters with emergency vehicles (ambulances, fire trucks, police) occur frequently in urban environments. E4 is not assigned because:
- Not every driving trip involves an emergency vehicle encounter
- Highway driving (significant operational time) has lower encounter rates
- Rural driving has substantially lower encounter rates

#### Controllability: C2 (Normally controllable — more than 90% of drivers can manage)

**Justification**: An approaching emergency vehicle is a developing situation with multiple cues available to the driver (and other road users):
- Visual cues: flashing lights, distinctive vehicle markings
- Auditory cues: siren audible to nearby human drivers even if AV fails to detect
- Other vehicle behavior: surrounding vehicles yielding provides cues
- C3 is not assigned because controllability is supported by the developing nature of the scenario and redundant environmental cues

#### ASIL Result

Per REF-001 (ASIL Determination Matrix): **S2 x E3 x C2 = ASIL B**

This is confirmed against ISO 26262-3 Table 4.

### 5.3 Fault Tolerant Time Interval (FTTI)

**FTTI: 500 ms**

**Justification**: The emergency vehicle yielding scenario has a larger time budget than forward collision scenarios (which have 100-150 ms FTTI). The 500 ms FTTI is justified because:

1. **Developing situation**: Emergency vehicles approach from behind or sides over multiple seconds; yielding is not a collision-imminent event
2. **Multiple detection opportunities**: At 20 Hz audio frame rate, 500 ms provides 10 audio frames for fault detection
3. **Secondary sensor**: Audio is a supplementary input; primary sensors (camera, LiDAR, radar) continue emergency vehicle detection independently during the FTTI window. Loss of audio reduces fusion confidence but does not eliminate detection capability.
4. **Vehicle dynamics**: Yielding involves lateral displacement (lane change) or controlled deceleration, both of which have response times on the order of seconds

The 500 ms FTTI must be confirmed by system-level timing analysis (see Open Items, Section 14).

---

## 6. Failure Modes & Freedom From Interference

### 6.1 Audio Input Failure Mode Catalog

The following failure modes are identified for the QM audio input path. Each failure mode is analyzed for its effect on the NDAS DRIVE AV emergency vehicle detection capability.

#### FM-01: Complete Loss of Audio Data

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | No audio frames received by AIQL |
| **Failure Mechanism** | Microphone hardware failure; codec power loss; driver crash; bus failure; memory allocation failure |
| **Effect on NDAS DRIVE AV** | Total loss of siren/horn detection; emergency vehicle detection relies solely on camera |
| **Detection Mechanism** | Frame reception timeout (TSR-AIQL-001), alive counter missing (TSR-AIQL-002); BIST loopback detection (TSR-AIQL-016); graceful degradation (TSR-AIQL-007) |
| **Safety Impact** | Potential violation of SG-03 if camera-only detection is insufficient |
| **Frequency Estimate** | Low (hardware failure rate ~500 FIT for complete path loss) |

#### FM-02: Audio Data Corruption

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio frame data is corrupted during transmission or processing |
| **Failure Mechanism** | Memory bit-flip (SEU); DMA error; buffer overrun in driver; EMI-induced data corruption on I2S bus |
| **Effect on NDAS DRIVE AV** | Corrupted audio may cause false siren detections (false positive) or mask real sirens (false negative) |
| **Detection Mechanism** | CRC-32 integrity check (TSR-AIQL-003), range validation (TSR-AIQL-005); graceful degradation (TSR-AIQL-007) |
| **Safety Impact** | False positive: unwarranted yielding (nuisance). False negative: violates SG-03 |
| **Frequency Estimate** | Medium (SEU rate ~200 FIT; EMI susceptibility dependent on shielding) |

#### FM-03: Audio Data Staleness

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio frames are received but contain outdated data (repeated or delayed frames) |
| **Failure Mechanism** | Driver buffer stall; scheduling priority inversion; DMA descriptor not updated; timestamp rollover |
| **Effect on NDAS DRIVE AV** | Stale audio data may show siren present when it has passed, or not-present when it has arrived |
| **Detection Mechanism** | Freshness check (TSR-AIQL-001), alive counter validation (TSR-AIQL-002), sequence counter monotonicity (TSR-AIQL-004); graceful degradation (TSR-AIQL-007) |
| **Safety Impact** | Delayed yielding response — violation of SG-03 FTTI |
| **Frequency Estimate** | Medium (scheduling issues ~100 FIT; increases under system load) |

#### FM-04: Audio Signal Out-of-Range

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio signal values exceed physical plausibility bounds |
| **Failure Mechanism** | Microphone bias failure; codec gain register corruption; ADC saturation; reference voltage drift |
| **Effect on NDAS DRIVE AV** | Clipped or offset audio distorts frequency content, leading to classifier errors |
| **Detection Mechanism** | Range validation (TSR-AIQL-005) — sample amplitude, DC offset, noise floor checks; BIST spectral match detects analog path degradation (TSR-AIQL-016) |
| **Safety Impact** | Classifier output unreliable — potential SG-03 violation |
| **Frequency Estimate** | Low (analog fault rate ~50 FIT; codec register corruption ~20 FIT) |

#### FM-05: False Negative (Missed Siren Detection)

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Real siren present but classifier reports no siren |
| **Failure Mechanism** | QM classifier systematic error; ambient noise masking; Doppler shift moving siren out of classifier frequency band; microphone sensitivity degradation |
| **Effect on NDAS DRIVE AV** | Reduced supplementary input for emergency vehicle detection — NDAS DRIVE AV continues to detect via primary sensors (camera, LiDAR, radar) |
| **Detection Mechanism** | **Not addressed by AIQL** — this is a classifier performance limitation, not a data integrity failure. As audio is a secondary sensor, FM-05 is mitigated at the system level by NDAS DRIVE AV multi-sensor fusion with primary sensors. The AIQL qualifies data integrity but does not validate classifier semantic correctness. |
| **Safety Impact** | Low at AIQL level — primary sensors provide independent emergency vehicle detection. Residual risk: reduced detection confidence in scenarios where audio would have provided early warning (e.g., emergency vehicle approaching from behind a visual obstruction). |
| **Frequency Estimate** | Medium-High (systematic failure; not quantifiable by FIT rate alone) |

#### FM-06: False Positive (Phantom Siren Detection)

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | No siren present but classifier reports siren detected |
| **Failure Mechanism** | QM classifier systematic error; environmental sounds resembling sirens (construction equipment, musical instruments, other vehicle horns); acoustic reflections causing phantom source |
| **Effect on NDAS DRIVE AV** | NDAS DRIVE AV receives false siren indication — effect is limited because audio is a secondary sensor and NDAS DRIVE AV's fusion weighs primary sensor inputs (camera, LiDAR, radar) for driving decisions |
| **Detection Mechanism** | **Not addressed by AIQL** — this is a classifier performance limitation, not a data integrity failure. As audio is a secondary sensor, FM-06 is mitigated at the system level by NDAS DRIVE AV multi-sensor fusion. Primary sensors provide independent confirmation; a false audio-only siren detection without corroborating primary sensor data will be downweighted by fusion. |
| **Safety Impact** | Low — unwarranted yielding requires corroboration from primary sensors in NDAS DRIVE AV's fusion architecture. Audio-only siren detection without primary sensor confirmation does not trigger yielding. |
| **Frequency Estimate** | Medium (systematic failure; dependent on acoustic environment) |

#### FM-07: Sequence Disorder

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Audio frames arrive out of sequence or with duplicated sequence numbers |
| **Failure Mechanism** | Multi-threaded driver race condition; DMA descriptor ring corruption; buffer management defect; interrupt priority inversion |
| **Effect on NDAS DRIVE AV** | Out-of-order frames disrupt temporal tracking of siren presence/absence transitions |
| **Detection Mechanism** | Sequence counter monotonicity check (TSR-AIQL-004) |
| **Safety Impact** | Delayed or incorrect siren detection state — potential SG-03 violation |
| **Frequency Estimate** | Low (software defect; ~10 FIT under normal scheduling) |

#### FM-08: Common Cause Failure

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | Multiple audio channels or both audio and AIQL fail simultaneously |
| **Failure Mechanism** | Shared power rail failure; shared clock source failure; SoC-wide thermal shutdown; common mode EMI affecting all audio channels; shared memory controller failure |
| **Effect on NDAS DRIVE AV** | Complete loss of qualified audio with potential AIQL integrity compromise |
| **Detection Mechanism** | Graceful degradation (TSR-AIQL-007), spatial FFI MPU (TSR-AIQL-012), temporal FFI watchdog (TSR-AIQL-013); diagnostic logging (TSR-AIQL-011) |
| **Safety Impact** | **Critical** — defeats both the QM element and the safety mechanism simultaneously |
| **Frequency Estimate** | Very Low (common cause beta factor ~2% of single-point failure rate) |

#### FM-09: Speaker Subsystem Failure

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | In-cabin speaker cannot produce BIST test signal |
| **Failure Mechanism** | Speaker driver IC failure; speaker coil open/short; codec DAC failure; wiring harness disconnection; amplifier fault |
| **Effect on NDAS DRIVE AV** | BIST cannot execute — audio path integrity cannot be verified; loss of BIST coverage for FM-01 and FM-04 |
| **Detection Mechanism** | BIST output monitoring (TSR-AIQL-015) — expected signal not detected at microphone within timeout; FF_BIST_SPEAKER flag set |
| **Safety Impact** | Loss of BIST diagnostic coverage — degrades confidence in audio path integrity. Triggers MRM after retry exhaustion (TSR-AIQL-019). |
| **Frequency Estimate** | Low (speaker failure rate ~100 FIT; amplifier failure ~50 FIT) |

#### FM-10: Acoustic Coupling Degradation

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | BIST spectral match fails due to degraded acoustic coupling between in-cabin speakers and microphones |
| **Failure Mechanism** | Physical obstruction (object placed over microphone); microphone membrane contamination; speaker baffle deformation; cabin seal change affecting acoustic path; microphone sensitivity drift |
| **Effect on NDAS DRIVE AV** | BIST indicates audio path degradation — correlation below threshold suggests microphone or acoustic path has changed, potentially affecting siren detection quality |
| **Detection Mechanism** | BIST spectral match correlation check (TSR-AIQL-016) — correlation < 0.85 or amplitude deviation > +/- 6 dB; FF_BIST_COUPLING flag set |
| **Safety Impact** | Indicates potential degradation of siren detection capability. Triggers MRM after retry exhaustion (TSR-AIQL-019). |
| **Frequency Estimate** | Medium (environmental contamination ~200 FIT; sensitivity drift ~50 FIT over lifetime) |

#### FM-11: BIST False Failure

| Attribute | Description |
|-----------|-------------|
| **Failure Mode** | BIST spectral match fails despite healthy audio path due to cabin noise interference |
| **Failure Mechanism** | Loud music playback during BIST window; passenger conversation masking BIST signal; HVAC blower noise at BIST frequencies; road noise during periodic BIST |
| **Effect on NDAS DRIVE AV** | False BIST failure triggers unnecessary MRM — availability impact (vehicle stops when audio path is actually functional) |
| **Detection Mechanism** | Retry mechanism (TSR-AIQL-017, TSR-AIQL-019) — 3 retries at startup, 2 retries during periodic BIST; signal-to-interference analysis during spectral match |
| **Safety Impact** | Availability impact only — false MRM is a nuisance but not a safety hazard. Retry mechanism reduces false failure probability. |
| **Frequency Estimate** | Medium (dependent on cabin noise environment; mitigated by BIST signal level at -20 dBFS and retry logic) |

### 6.2 Freedom From Interference (FFI) Analysis

FFI analysis ensures that failures in the QM audio subsystem cannot propagate into and corrupt the ASIL B AIQL or the ASIL-rated NDAS DRIVE AV system. Three FFI dimensions are analyzed per ISO 26262-9.

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
| **Requirement** | When the AIQL qualification state transitions to NOT_QUALIFIED, the AIQL shall: (a) Output qualification_status = NOT_QUALIFIED to NDAS DRIVE AV; (b) Clear the audio_data and classifier_output fields (zero-fill); (c) Continue outputting frames at 20 Hz to maintain alive signaling; (d) Set all applicable failure_flags bits. NDAS DRIVE AV shall exclude audio from sensor fusion and continue emergency vehicle detection using primary sensors (camera, LiDAR, radar) upon receiving NOT_QUALIFIED status. When NOT_QUALIFIED is caused by confirmed BIST failure (FF_BIST_FAIL, FF_BIST_SPEAKER, or FF_BIST_COUPLING set after retry exhaustion), the AIQL shall additionally request MRM transition per TSR-AIQL-019. |
| **ASIL** | B |
| **Rationale** | Graceful degradation ensures that audio failures result in a deterministic exclusion of audio from NDAS DRIVE AV sensor fusion rather than uncontrolled behavior. Zero-filling audio data prevents NDAS DRIVE AV from using corrupted or stale data. Continued 20 Hz output ensures NDAS DRIVE AV can distinguish "audio not qualified" from "AIQL has crashed." As audio is a secondary sensor, the transition to primary-sensor-only operation is a minimal degradation. |
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

### TSR-AIQL-015: BIST Reference Signal Generation

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-015 |
| **Requirement** | The AIQL shall generate a logarithmic sine sweep reference signal from 500 Hz to 3000 Hz at -20 dBFS amplitude for BIST loopback verification. The signal shall be output to the in-cabin speakers via the codec DAC interface (Section 4.2.5). The signal shall have total harmonic distortion (THD) < 1% and frequency accuracy within +/- 2% of the specified sweep profile. |
| **ASIL** | B |
| **Rationale** | The BIST reference signal must cover the siren frequency range (500-3000 Hz) to verify the audio path across all frequencies relevant to emergency vehicle detection. The -20 dBFS amplitude is chosen to be clearly audible above typical cabin ambient noise (~40-60 dB SPL) while remaining below uncomfortable listening levels. Logarithmic sweep provides equal energy per octave, matching human hearing perception and siren frequency distribution. |
| **Failure Mode Addressed** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) |
| **Verification** | FI-10, unit test |
| **Acceptance Criterion** | Signal generation within specified frequency range, amplitude, and THD limits; sweep profile matches reference within +/- 2% frequency accuracy. |

### TSR-AIQL-016: BIST Spectral Match Verification

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-016 |
| **Requirement** | The AIQL shall capture the microphone response to the BIST reference signal and perform spectral match verification. The verification shall compute: (a) Normalized cross-correlation between expected and received spectral envelopes — pass threshold: correlation >= 0.85; (b) Amplitude deviation per 1/3-octave band — pass threshold: deviation within +/- 6 dB of reference. Both criteria must be met for BIST pass. Failure of either criterion shall set the appropriate failure flag (FF_BIST_FAIL, FF_BIST_COUPLING). |
| **ASIL** | B |
| **Rationale** | Spectral match verification confirms that the complete acoustic-to-digital path (speaker → air → microphone → codec → digital) preserves the frequency characteristics needed for siren detection. The 0.85 correlation threshold allows for normal cabin acoustic variation while detecting significant path degradation. The +/- 6 dB amplitude tolerance per 1/3-octave band detects frequency-selective faults (e.g., microphone resonance shift, speaker cone damage) while accommodating cabin acoustic transfer function variation. |
| **Failure Mode Addressed** | FM-01 (Loss), FM-04 (Out-of-Range), FM-10 (Coupling Degradation) |
| **Verification** | FI-11, unit test |
| **Acceptance Criterion** | Correct pass/fail determination for all test vectors; zero false passes for injected path degradation exceeding thresholds; false failure rate < 0.1% under nominal cabin conditions. |

### TSR-AIQL-017: Startup BIST Precondition

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-017 |
| **Requirement** | The AIQL shall execute a full startup BIST (2000 ms sine sweep) before transitioning from NOT_QUALIFIED to QUALIFIED state on system initialization. The startup BIST must pass before any audio data is qualified for NDAS DRIVE AV. If the startup BIST fails, the AIQL shall retry up to 3 times. If all 3 retries fail, the AIQL shall remain in NOT_QUALIFIED state and request MRM transition per TSR-AIQL-019. |
| **ASIL** | B |
| **Rationale** | Startup BIST ensures the audio path is verified before any audio data influences NDAS DRIVE AV sensor fusion. The 2000 ms sweep duration provides comprehensive frequency coverage. Three retries accommodate transient interference during vehicle startup (e.g., door closing, engine start noise). Total startup BIST budget: ~8.2 seconds worst case (4 attempts x 2050 ms). |
| **Failure Mode Addressed** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) |
| **Verification** | FI-12, unit test |
| **Acceptance Criterion** | No transition to QUALIFIED without BIST pass; all 3 retries executed on persistent failure; MRM requested after retry exhaustion. |

### TSR-AIQL-018: Periodic BIST Scheduling

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-018 |
| **Requirement** | The AIQL shall execute a periodic BIST (200 ms abbreviated sweep burst) every 60 seconds during QUALIFIED operation. During the periodic BIST window, the AIQL shall: (a) Subtract the known BIST signal from the audio data before forwarding to NDAS DRIVE AV (signal cancellation); (b) Perform spectral match verification per TSR-AIQL-016 on the microphone capture. Signal cancellation shall achieve >= 30 dB attenuation of the BIST signal in the forwarded audio stream. |
| **ASIL** | B |
| **Rationale** | Periodic BIST detects audio path degradation that develops during driving (e.g., microphone contamination, connector vibration loosening, thermal drift). The 60-second interval balances diagnostic coverage against cabin audio intrusion. The 200 ms burst is shorter than the startup sweep but covers the critical siren frequency range. Signal cancellation prevents the BIST signal from reaching NDAS DRIVE AV and causing false siren detections. The 30 dB cancellation target reduces the BIST signal below the classifier noise floor. |
| **Failure Mode Addressed** | FM-10 (Coupling Degradation), FM-11 (BIST False Failure) |
| **Verification** | FI-13, unit test |
| **Acceptance Criterion** | BIST executes within +/- 5 seconds of scheduled interval; signal cancellation >= 30 dB measured at AIQL output; no false siren detections by classifier during BIST window. |

### TSR-AIQL-019: MRM Transition on Confirmed BIST Failure

| Attribute | Value |
|-----------|-------|
| **ID** | TSR-AIQL-019 |
| **Requirement** | Upon confirmed BIST failure (startup: 3 consecutive failures; periodic: 2 consecutive failures), the AIQL shall: (a) Set qualification_status = NOT_QUALIFIED with FF_BIST_FAIL (and FF_BIST_SPEAKER or FF_BIST_COUPLING as applicable); (b) Transmit an MRM request to the vehicle safety manager via a dedicated safety communication channel; (c) Log the BIST failure details and MRM request to the diagnostic log (TSR-AIQL-011). The MRM request shall trigger a controlled vehicle stop (Minimal Risk Maneuver). The MRM_REQUESTED state is terminal — recovery requires system restart. |
| **ASIL** | B |
| **Rationale** | BIST failure indicates a fundamental hardware or acoustic path degradation that cannot be resolved by software. Unlike passive data integrity checks (freshness, CRC), which detect transient faults, BIST failure reveals structural degradation of the audio capture chain. MRM is appropriate because: (1) the audio path cannot be trusted for siren detection; (2) the degradation is likely persistent (not transient); (3) continued driving without verified audio path integrity violates the safety concept assumptions. Two retries for periodic BIST (vs. 3 for startup) reflect the higher confidence that a failure during driving represents a real fault rather than transient noise. |
| **Failure Mode Addressed** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) |
| **Verification** | FI-14, unit test |
| **Acceptance Criterion** | MRM requested within 500 ms of confirmed BIST failure; no MRM request before retry exhaustion; MRM_REQUESTED state is terminal (no automatic recovery). |

### 7.1 TSR Summary Table

| TSR ID | Short Name | ASIL | Failure Modes | FFI Type |
|--------|-----------|------|---------------|----------|
| TSR-AIQL-001 | Freshness Check | B | FM-01, FM-03 | Communication |
| TSR-AIQL-002 | Alive Counter | B | FM-01, FM-03 | Communication |
| TSR-AIQL-003 | CRC-32 Integrity | B | FM-02 | Communication |
| TSR-AIQL-004 | Sequence Counter | B | FM-07 | Communication |
| TSR-AIQL-005 | Range Validation | B | FM-04 | Communication |
| TSR-AIQL-007 | Graceful Degradation | B | FM-01, FM-02, FM-03, FM-08, FM-09, FM-10 | — |
| TSR-AIQL-008 | Recovery Criteria | B | All | — |
| TSR-AIQL-009 | WCET | B | Temporal FFI | Temporal |
| TSR-AIQL-010 | ASIL B Process | B | All | — |
| TSR-AIQL-011 | Diagnostic Logging | QM | All | — |
| TSR-AIQL-012 | Spatial FFI (MPU) | B | FM-08, Spatial FFI | Spatial |
| TSR-AIQL-013 | Temporal FFI (Watchdog) | B | FM-08, Temporal FFI | Temporal |
| TSR-AIQL-015 | BIST Signal Generation | B | FM-09, FM-10 | — |
| TSR-AIQL-016 | BIST Spectral Match | B | FM-01, FM-04, FM-10 | — |
| TSR-AIQL-017 | Startup BIST | B | FM-09, FM-10 | — |
| TSR-AIQL-018 | Periodic BIST | B | FM-10, FM-11 | — |
| TSR-AIQL-019 | MRM on BIST Failure | B | FM-09, FM-10 | — |

---

## 8. ASIL Decomposition

### 8.1 Decomposition Argument

The ASIL B safety integrity for SG-03 is achieved through ASIL decomposition per ISO 26262-9 Clause 5, applied at the boundary between the QM audio subsystem and the ASIL B AIQL.

```
Safety Goal SG-03: ASIL B
    |
    +--- ASIL B (AIQL) — Safety mechanism qualifying audio input + BIST
    |
    +--- QM (Audio HW) — In-cabin microphones, codec, amplifiers
    |
    +--- QM (Audio SW) — Audio driver, DSP firmware, siren classifier
    |
    +--- QM (Speaker HW) — In-cabin speakers, DAC, speaker amplifier
```

**Decomposition**: ASIL B = ASIL B(AIQL) + QM(Audio HW) + QM(Audio SW) + QM(Speaker HW)

This decomposition is valid under ISO 26262-9 Clause 5 because:

1. **The ASIL B element (AIQL) implements all safety mechanisms necessary to detect and mitigate data integrity failures in the QM elements** — as enumerated in Section 7 (TSR-AIQL-001 through TSR-AIQL-005, TSR-AIQL-007 through TSR-AIQL-019). Classifier correctness (FM-05, FM-06) is addressed at the system level by NDAS DRIVE AV multi-sensor fusion with primary sensors, not by the AIQL.

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
| Sufficient safety mechanisms in higher-ASIL element | Yes — 16 ASIL B TSRs + 1 QM TSR covering all data integrity failure modes (Section 7); classifier correctness addressed at system level |
| QM element Assumptions of Use documented | Yes — Section 9 |

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

### AoU-009: In-Cabin Microphone Specifications

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-009 |
| **Assumption** | The in-cabin microphone array shall consist of at least 4 microphones mounted within the vehicle cabin with: (a) Sensitivity >= -38 dBV/Pa; (b) Signal-to-noise ratio >= 65 dB(A); (c) Frequency response covering 500 Hz to 3000 Hz (siren frequency range) with <= 3 dB variation; (d) Operating temperature range -40 C to +85 C; (e) Placement providing coverage of cabin acoustic space for BIST loopback reception (minimum 2 microphones with line-of-sight to at least one in-cabin speaker). |
| **Rationale** | The siren classifier performance depends on adequate audio quality. In-cabin microphone placement means external siren audio is attenuated by cabin insulation (typically 20-30 dB); the microphone specifications must support detection of attenuated external sounds. Additionally, microphone placement must support BIST loopback verification by ensuring adequate acoustic coupling to in-cabin speakers. |
| **TSR Dependency** | TSR-AIQL-005 (noise floor check), TSR-AIQL-016 (BIST spectral match) |
| **Verification of AoU** | Microphone specification review; in-cabin acoustic characterization test; BIST loopback validation at design placement |

### AoU-010: Self-Diagnostic Reporting

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-010 |
| **Assumption** | The QM audio subsystem shall provide a self-diagnostic status byte in each audio frame indicating: (a) Microphone connectivity (all channels active); (b) Codec operational status; (c) Driver initialization status. A value of 0x00 indicates all-OK; non-zero values indicate specific faults per a defined encoding. |
| **Rationale** | While the AIQL performs independent qualification checks, QM self-diagnostic information provides an additional indicator for early fault detection and diagnostic logging (TSR-AIQL-011). This is a defense-in-depth measure — the AIQL does not rely solely on this information for safety decisions. |
| **TSR Dependency** | TSR-AIQL-011 (diagnostic logging) |
| **Verification of AoU** | Fault injection in QM subsystem; verify diagnostic byte correctly reports induced faults |

### AoU-011: Speaker Availability and Specifications

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-011 |
| **Assumption** | The vehicle shall provide at least one in-cabin speaker available for BIST signal output with: (a) Frequency response covering 500 Hz to 3000 Hz; (b) Sound pressure level >= 70 dB SPL at microphone position when driven at -20 dBFS; (c) Total harmonic distortion (THD) < 5% within the BIST frequency range; (d) Speaker accessible via codec DAC interface without requiring entertainment system to be active. |
| **Rationale** | The BIST loopback requires a speaker capable of reproducing the sine sweep test signal across the siren frequency range. The 70 dB SPL minimum ensures adequate SNR for spectral match verification against typical cabin ambient noise (40-60 dB SPL). The speaker must be independently addressable — BIST must function regardless of entertainment system state. |
| **TSR Dependency** | TSR-AIQL-015 (BIST signal generation), TSR-AIQL-016 (BIST spectral match) |
| **Verification of AoU** | Speaker specification review; acoustic output measurement at microphone positions; BIST signal reproduction quality test |

### AoU-012: Speaker-to-Microphone Acoustic Coupling

| Attribute | Value |
|-----------|-------|
| **ID** | AoU-012 |
| **Assumption** | The acoustic coupling between the BIST speaker and the in-cabin microphone array shall provide: (a) Signal-to-noise ratio >= 20 dB for the BIST signal at the microphone position under worst-case cabin ambient noise conditions (windows closed, HVAC maximum); (b) Frequency response variation <= 10 dB across 500-3000 Hz range for the speaker-to-microphone acoustic path; (c) Stable acoustic coupling (variation < 3 dB) across operating temperature range and normal cabin configurations. |
| **Rationale** | The BIST spectral match algorithm (TSR-AIQL-016) requires sufficient SNR to distinguish the test signal from cabin noise. The 20 dB SNR minimum ensures reliable spectral correlation even in noisy cabin conditions. Frequency response and stability requirements ensure the BIST reference calibration remains valid across operating conditions. |
| **TSR Dependency** | TSR-AIQL-016 (BIST spectral match), TSR-AIQL-018 (periodic BIST) |
| **Verification of AoU** | In-cabin acoustic characterization; BIST SNR measurement across temperature and HVAC operating points; long-term stability validation |

### 9.1 AoU Summary Table

| AoU ID | Short Name | TSR Dependency | Verification Method |
|--------|-----------|----------------|---------------------|
| AoU-001 | CRC-32 Generation | TSR-AIQL-003 | Unit test, integration test |
| AoU-002 | Sequence Counter | TSR-AIQL-004 | Unit test, wraparound test |
| AoU-003 | Timestamp | TSR-AIQL-001 | Accuracy measurement |
| AoU-004 | Alive Counter | TSR-AIQL-002 | Unit test, stress test |
| AoU-005 | Frame Rate | TSR-AIQL-001, TSR-AIQL-008, TSR-AIQL-009 | Long-duration measurement |
| AoU-006 | Memory Isolation | TSR-AIQL-012 | MPU fault injection, static analysis |
| AoU-007 | CPU Budget | TSR-AIQL-013 | WCET analysis, runtime monitoring |
| AoU-008 | Signal Range | TSR-AIQL-005 | Format verification, boundary values |
| AoU-009 | In-Cabin Microphone Specs | TSR-AIQL-005, TSR-AIQL-016 | Spec review, acoustic test, BIST validation |
| AoU-010 | Self-Diagnostic | TSR-AIQL-011 | Fault injection |
| AoU-011 | Speaker Availability | TSR-AIQL-015, TSR-AIQL-016 | Spec review, acoustic measurement |
| AoU-012 | Acoustic Coupling | TSR-AIQL-016, TSR-AIQL-018 | Acoustic characterization, stability test |

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
              (passive faults        | Confirmed BIST failure
               only)                 | (retry exhaustion)
                                     v
                          +====================+
                          |  MRM_REQUESTED     |
                          |  (Terminal State)  |
                          +====================+
                           No automatic recovery
                           Requires system restart
```

### 10.2 State Definitions

#### QUALIFIED (Normal Operation)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | All checks passing; sufficient consecutive valid frames received (10 from DEGRADED, 20 from NOT_QUALIFIED); startup BIST passed |
| **Behavior** | Audio data and classifier output passed through to NDAS DRIVE AV with qualification_status = QUALIFIED |
| **NDAS DRIVE AV Action** | Multi-sensor fusion with audio as supplementary input for emergency vehicle detection (nominal mode) |
| **Output** | Full audio frame + classifier output + qualification_status = QUALIFIED |

#### DEGRADED (Monitoring)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | Any single qualification check failure while in QUALIFIED state |
| **Behavior** | Audio data still passed through to NDAS DRIVE AV with qualification_status = DEGRADED; failure_flags indicate which checks failed; AIQL monitors for recovery or further degradation |
| **NDAS DRIVE AV Action** | Reduced confidence in audio channel; increase weight on primary sensors (camera, LiDAR, radar) for emergency vehicle detection |
| **Output** | Full audio frame + classifier output + failure_flags + qualification_status = DEGRADED |
| **Escalation** | Transition to NOT_QUALIFIED after 3 consecutive frame failures OR frame reception timeout > 150 ms |

#### NOT_QUALIFIED (Safe State)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | 3 consecutive frame failures in DEGRADED state; frame reception timeout > 150 ms; watchdog trigger; MPU violation detected |
| **Behavior** | Audio data and classifier output zeroed out; qualification_status = NOT_QUALIFIED; continue 20 Hz output for liveness signaling |
| **NDAS DRIVE AV Action** | Primary-sensor-only emergency vehicle detection; audio excluded from fusion. Primary sensors (camera, LiDAR, radar) continue normal operation. |
| **Output** | Zero-filled audio + zero-filled classifier + all failure_flags set + qualification_status = NOT_QUALIFIED |
| **Recovery** | Transition to DEGRADED after 20 consecutive valid frames (1000 ms) — only for passive fault recovery. If NOT_QUALIFIED was caused by confirmed BIST failure, recovery is not possible — state transitions to MRM_REQUESTED. |

#### MRM_REQUESTED (Terminal State)

| Attribute | Value |
|-----------|-------|
| **Entry Condition** | Confirmed BIST failure: startup BIST fails 3 consecutive times (TSR-AIQL-017); periodic BIST fails 2 consecutive times (TSR-AIQL-019) |
| **Behavior** | AIQL outputs NOT_QUALIFIED status with BIST failure flags; MRM request transmitted to vehicle safety manager; diagnostic log captures BIST failure details |
| **NDAS DRIVE AV Action** | Audio excluded from fusion; vehicle safety manager initiates controlled stop (Minimal Risk Maneuver) |
| **Output** | Zero-filled audio + zero-filled classifier + BIST failure flags + qualification_status = NOT_QUALIFIED + MRM request signal |
| **Recovery** | **None** — MRM_REQUESTED is a terminal state. Recovery requires full system restart and successful startup BIST. This reflects the persistent nature of the underlying hardware or acoustic path failure. |

### 10.3 Fallback and MRM Strategy

When the AIQL enters a degraded or failed state, two distinct strategies apply depending on the failure type:

| Aspect | Passive Fault (NOT_QUALIFIED) | BIST Failure (MRM_REQUESTED) |
|--------|-------------------------------|------------------------------|
| **Trigger** | Data integrity check failures (freshness, CRC, sequence, range, timeout) | Confirmed BIST loopback failure after retry exhaustion |
| **Failure Nature** | Potentially transient (software glitch, scheduling issue, transient EMI) | Likely persistent (hardware degradation, acoustic path obstruction) |
| **Vehicle Action** | Continue driving with primary sensors only; audio excluded from fusion | Controlled stop (MRM) — vehicle brought to safe standstill |
| **Detection Sources** | Camera, LiDAR, radar continue nominal operation | Camera, LiDAR, radar continue during MRM maneuver |
| **Recovery** | Automatic — AIQL recovers when 20 consecutive valid frames received | Manual — requires system restart and successful startup BIST |
| **Justification** | Audio is a secondary sensor; transient loss is tolerable with primary sensors active | BIST failure indicates structural audio path degradation; continued operation violates safety concept assumptions |
| **Duration** | Until recovery or end of drive cycle | Until vehicle stops and system is restarted |
| **Driver Notification** | HMI: "Audio detection system degraded" | HMI: "Audio system fault — vehicle stopping" |

**BIST Failure-to-MRM Timing Budget**:

| Phase | Duration | Running Total | Activity |
|-------|----------|---------------|----------|
| BIST failure detected | 0 ms | 0 ms | Periodic BIST spectral match fails |
| First retry | ~250 ms | 250 ms | Abbreviated sweep (200 ms) + analysis (50 ms) |
| Second retry | ~250 ms | 500 ms | Abbreviated sweep (200 ms) + analysis (50 ms) |
| MRM request transmitted | <= 100 ms | 600 ms | AIQL sends MRM to vehicle safety manager |
| Vehicle safety manager processing | <= 200 ms | 800 ms | MRM path planning initiated |
| Vehicle deceleration begins | <= 200 ms | 1000 ms | Brake application starts |
| Vehicle stop (from 60 km/h) | ~30000 ms | ~31000 ms | Controlled deceleration to standstill |
| **Total: fault to vehicle stop** | | **~31 seconds** | |

### 10.4 FTTI Budget Allocation

The 500 ms FTTI is allocated across the degradation sequence:

| Phase | Duration | Running Total | Activity |
|-------|----------|---------------|----------|
| Fault occurrence | 0 ms | 0 ms | Audio path fault occurs |
| Fault detection | <= 100 ms | 100 ms | AIQL detects fault (2 frame periods) |
| State transition to DEGRADED | <= 50 ms | 150 ms | AIQL sets DEGRADED status |
| State transition to NOT_QUALIFIED | <= 50 ms | 200 ms | If fault persists, AIQL sets NOT_QUALIFIED |
| NDAS DRIVE AV fusion update | <= 100 ms | 300 ms | NDAS DRIVE AV excludes audio from fusion, continues with primary sensors |
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
| **System Test** | Full vehicle or vehicle-representative bench | End-to-end audio path including microphones, codec, driver, AIQL, and NDAS DRIVE AV interface | ISO 26262-4, Table 9 |

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

#### FI-10: Speaker Failure Injection

| Attribute | Value |
|-----------|-------|
| **ID** | FI-10 |
| **Objective** | Verify AIQL detects speaker subsystem failures preventing BIST execution |
| **Failure Modes** | FM-09 (Speaker Subsystem Failure) |
| **TSRs Verified** | TSR-AIQL-015 (BIST Signal Generation), TSR-AIQL-019 (MRM on BIST Failure) |
| **Injection Method** | (a) Disconnect speaker output (open circuit); (b) Short speaker output; (c) Disable codec DAC; (d) Corrupt BIST signal output (inject noise) |
| **Pass Criteria** | FF_BIST_SPEAKER set within BIST timeout; MRM requested after retry exhaustion; no false speaker failure flags during normal operation |
| **Test Count** | 8 test cases (4 injection types x 2 BIST phases: startup, periodic) |

#### FI-11: Spectral Match Degradation

| Attribute | Value |
|-----------|-------|
| **ID** | FI-11 |
| **Objective** | Verify AIQL detects acoustic coupling degradation via BIST spectral match |
| **Failure Modes** | FM-10 (Acoustic Coupling Degradation) |
| **TSRs Verified** | TSR-AIQL-016 (BIST Spectral Match Verification) |
| **Injection Method** | (a) Attenuate received signal by 10 dB, 20 dB, 30 dB (gradual mic sensitivity loss); (b) Notch filter at 1 kHz, 2 kHz (frequency-selective fault); (c) Add broadband noise at varying SNR levels; (d) Phase shift received signal (acoustic path change) |
| **Pass Criteria** | FF_BIST_COUPLING set when correlation < 0.85 or amplitude deviation > +/- 6 dB; correct pass for signals within tolerance; threshold behavior verified at boundary |
| **Test Count** | 12 test cases (4 injection types x 3 severity levels) |

#### FI-12: Startup BIST Blocking

| Attribute | Value |
|-----------|-------|
| **ID** | FI-12 |
| **Objective** | Verify AIQL blocks QUALIFIED transition until startup BIST passes |
| **Failure Modes** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) |
| **TSRs Verified** | TSR-AIQL-017 (Startup BIST Precondition) |
| **Injection Method** | (a) Fail startup BIST 1 time then pass (verify retry); (b) Fail startup BIST 2 times then pass; (c) Fail startup BIST 3 times (verify MRM request); (d) Inject transient noise during startup BIST |
| **Pass Criteria** | No QUALIFIED transition without BIST pass; correct retry count (up to 3); MRM requested after 3 consecutive failures; total startup time within budget |
| **Test Count** | 8 test cases (4 injection types x 2 failure patterns) |

#### FI-13: Periodic BIST During Driving

| Attribute | Value |
|-----------|-------|
| **ID** | FI-13 |
| **Objective** | Verify periodic BIST execution and signal cancellation during QUALIFIED operation |
| **Failure Modes** | FM-10 (Coupling Degradation), FM-11 (BIST False Failure) |
| **TSRs Verified** | TSR-AIQL-018 (Periodic BIST Scheduling) |
| **Injection Method** | (a) Inject cabin noise at various levels during periodic BIST; (b) Degrade acoustic coupling mid-drive; (c) Verify signal cancellation by monitoring NDAS DRIVE AV input during BIST; (d) Verify BIST interval timing under various system loads |
| **Pass Criteria** | Periodic BIST executes within +/- 5 seconds of 60-second interval; signal cancellation >= 30 dB; no false siren detections during BIST window; correct failure detection when coupling degrades |
| **Test Count** | 12 test cases (4 injection types x 3 operating conditions) |

#### FI-14: BIST-Triggered MRM

| Attribute | Value |
|-----------|-------|
| **ID** | FI-14 |
| **Objective** | Verify AIQL correctly requests MRM on confirmed BIST failure and enters terminal state |
| **Failure Modes** | FM-09 (Speaker Failure), FM-10 (Coupling Degradation) |
| **TSRs Verified** | TSR-AIQL-019 (MRM Transition on Confirmed BIST Failure) |
| **Injection Method** | (a) Persistent speaker failure during periodic BIST (2 retries then MRM); (b) Persistent coupling degradation during periodic BIST; (c) Verify MRM_REQUESTED is terminal (no recovery without restart); (d) Verify MRM request timing (within 500 ms of confirmed failure); (e) Verify MRM during startup BIST (3 retries then MRM) |
| **Pass Criteria** | MRM requested after exactly 2 periodic retries or 3 startup retries; MRM request within 500 ms; MRM_REQUESTED state is terminal; diagnostic log contains BIST failure details |
| **Test Count** | 10 test cases (5 injection types x 2 BIST phases) |

### 11.4 Fault Injection Campaign Summary

| FI ID | Failure Modes | TSRs Verified | Test Count | Level |
|-------|---------------|---------------|------------|-------|
| FI-01 | FM-01, FM-03 | TSR-AIQL-001, -002 | 12 | HiL |
| FI-02 | FM-03 | TSR-AIQL-001 | 8 | HiL |
| FI-03 | FM-02 | TSR-AIQL-003 | 20 | HiL |
| FI-04 | FM-07 | TSR-AIQL-004 | 10 | HiL |
| FI-05 | FM-04 | TSR-AIQL-005 | 18 | HiL |
| FI-07 | All (data integrity) | TSR-AIQL-007, -008 | 15 | HiL |
| FI-08 | FM-08 | TSR-AIQL-012 | 8 | HiL |
| FI-09 | FM-08 | TSR-AIQL-013 | 8 | HiL |
| FI-10 | FM-09 | TSR-AIQL-015, -019 | 8 | HiL |
| FI-11 | FM-10 | TSR-AIQL-016 | 12 | HiL |
| FI-12 | FM-09, FM-10 | TSR-AIQL-017 | 8 | HiL |
| FI-13 | FM-10, FM-11 | TSR-AIQL-018 | 12 | HiL |
| FI-14 | FM-09, FM-10 | TSR-AIQL-019 | 10 | HiL |
| **Total** | | | **149** | |

**Note**: FM-05 (False Negative) and FM-06 (False Positive) are classifier performance limitations addressed at the system level by NDAS DRIVE AV multi-sensor fusion with primary sensors. They are not covered by the AIQL fault injection campaign.

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
| **Hazardous Behavior** | Failure to detect approaching emergency vehicle — violates SG-03 |
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
| **Risk Reduction** | **System-level mitigation**: NDAS DRIVE AV multi-sensor fusion weighs audio as a secondary input. A false siren detection from audio alone, without corroborating primary sensor evidence (flashing lights on camera, emergency vehicle shape on LiDAR), will be downweighted by fusion and will not trigger a yielding maneuver. |
| **AIQL Response** | AIQL remains in QUALIFIED state — the audio I/O path is functioning correctly. The classifier output passes all range checks (probability within [0.0, 1.0]). False positive detection is a classifier performance issue addressed at the system level. |
| **Residual Risk** | Low — audio is a secondary sensor. False audio-only siren detections do not independently trigger yielding; primary sensor corroboration is required by NDAS DRIVE AV's fusion architecture. |

### TC-AUDIO-03: Doppler Shift

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | Emergency vehicle approaching at high relative speed causes Doppler shift that moves siren fundamental frequency outside the classifier's trained frequency band |
| **Functional Insufficiency** | QM siren classifier trained on stationary siren frequency profiles; Doppler-shifted frequencies may fall outside recognition band |
| **Hazardous Behavior** | Delayed or missed siren detection when emergency vehicle approaches at high closing speed — potential SG-03 violation |
| **Scenario** | Emergency vehicle approaching head-on at 120 km/h relative speed; siren fundamental at 1000 Hz shifted to ~1100 Hz; if classifier frequency band is narrowly tuned, shifted frequency may reduce detection confidence |
| **Risk Reduction** | **System-level mitigation**: Primary sensors (camera, LiDAR, radar) detect emergency vehicles by visual/shape/motion characteristics unaffected by Doppler shift. Classifier AoU should specify Doppler-robust frequency band (siren wail sweeps 500-1600 Hz; Doppler shift at max relative speed is ~10%). SOTIF validation testing with Doppler-shifted audio. |
| **AIQL Response** | AIQL cannot detect Doppler-related misclassification — this is a classifier performance limitation, not a data integrity failure. Diagnostic logging (TSR-AIQL-011) captures events for post-drive analysis. |
| **Residual Risk** | Low — audio is a secondary sensor. Doppler shift at typical urban speeds (< 60 km/h relative) is < 5%, well within siren wail sweep range. Primary sensors unaffected by acoustic Doppler. |

### TC-AUDIO-04: In-Cabin Microphone Acoustic Attenuation

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | Vehicle cabin acoustic insulation (windows closed, premium sound deadening, active noise cancellation) attenuates external siren sound before reaching in-cabin microphones by 20-30 dB |
| **Functional Insufficiency** | In-cabin microphone placement inherently reduces external siren SNR compared to exterior mounting; cabin insulation acts as a low-pass filter attenuating high-frequency siren harmonics |
| **Hazardous Behavior** | Reduced siren detection range — emergency vehicle must be significantly closer before detection occurs, reducing available yielding time |
| **Scenario** | Premium vehicle with laminated acoustic glass and ANC active; windows closed; exterior siren at 200 m (siren source at 123 dB(A) at 1 m, attenuated to ~70 dB(A) at 200 m in free field); cabin insulation attenuates by 25 dB to ~45 dB(A) at in-cabin microphone; cabin ambient noise at 50 dB(A) — siren SNR is negative, detection not possible until closer range |
| **Risk Reduction** | **System-level mitigation**: Primary sensors (camera, LiDAR, radar) detect emergency vehicles by visual/shape/motion characteristics unaffected by cabin acoustic insulation. AoU-009 specifies minimum in-cabin microphone sensitivity and SNR. BIST verifies the acoustic-to-digital path integrity but cannot compensate for cabin attenuation of external sounds (BIST verifies the internal speaker-to-mic path, not the external sound-to-mic path). Classifier may require training specifically for attenuated in-cabin siren profiles. |
| **AIQL Response** | AIQL cannot distinguish low-amplitude attenuated siren from genuine absence of siren. BIST confirms the audio path hardware is functional; the attenuation is an inherent physical characteristic of in-cabin placement, not a fault. Range validation (TSR-AIQL-005) noise floor check verifies microphone connectivity but not external sound coupling. |
| **Residual Risk** | **Medium** — audio is a secondary sensor, but in-cabin placement significantly reduces the effective siren detection range compared to exterior mounting. Detection range reduction estimated at 50-70% (from ~200 m to ~60-100 m for typical siren). Primary sensors mitigate but the reduced audio early-warning capability is a meaningful loss in scenarios where audio would have provided first detection (e.g., emergency vehicle behind visual obstruction). This residual risk is accepted because: (1) primary sensors maintain independent detection capability; (2) in-cabin placement enables BIST verification not possible with exterior microphones; (3) in-cabin placement eliminates weather/road noise exposure issues. |

### TC-AUDIO-05: BIST Cabin Noise Interference

| Attribute | Value |
|-----------|-------|
| **Triggering Condition** | High cabin noise during BIST window (music playback, passenger conversation, HVAC at maximum, road noise on rough surface) masks the BIST reference signal at microphones |
| **Functional Insufficiency** | BIST spectral match algorithm cannot reliably distinguish BIST signal from high-level cabin noise when SNR < 10 dB |
| **Hazardous Behavior** | False BIST failure triggers unnecessary MRM — availability impact (vehicle performs controlled stop when audio path is actually functional) |
| **Scenario** | Periodic BIST executes during highway driving; passengers playing music at 85 dB(A) via in-cabin speakers; BIST signal at -20 dBFS produces ~70 dB SPL at microphone; music energy in BIST frequency range masks the sweep signal; spectral correlation drops below 0.85 threshold |
| **Risk Reduction** | Retry mechanism (TSR-AIQL-019) — 2 retries for periodic BIST reduce false failure probability (noise conditions likely to change between retries). BIST signal level (-20 dBFS) designed to exceed typical cabin noise. Future enhancement: BIST scheduling could avoid windows with detected high ambient noise (see OI-14). Signal cancellation technique (TSR-AIQL-018) partially mitigates by subtracting known BIST signal before spectral analysis. |
| **AIQL Response** | If noise causes BIST failure, retry mechanism provides second chance. If both retries fail, MRM is requested — this is a conservative (safe) response. The false MRM rate is an availability concern, not a safety concern. |
| **Residual Risk** | Low for safety (false MRM is safe, not hazardous). **Medium for availability** — false BIST failures reduce system uptime. Target: false BIST failure rate < 1 per 1000 driving hours (see OI-11). |

### 12.1 SOTIF Summary

| TC ID | Triggering Condition | Primary Mitigation | AIQL Mechanism | Residual Risk |
|-------|---------------------|-------------------|----------------|---------------|
| TC-AUDIO-01 | Ambient noise masking | Primary sensors (system-level) | TSR-AIQL-005 (saturation detection) | Low |
| TC-AUDIO-02 | Siren-like sounds | Primary sensors + fusion weighting (system-level) | None (classifier performance) | Low |
| TC-AUDIO-03 | Doppler shift | Primary sensors (system-level) | None (classifier performance) | Low |
| TC-AUDIO-04 | In-cabin mic attenuation | Primary sensors + in-cabin mic specs | AoU-009, BIST (path verification only) | Medium |
| TC-AUDIO-05 | BIST cabin noise interference | Retry mechanism + signal level design | TSR-AIQL-019 (retries), TSR-AIQL-018 (cancellation) | Low (safety) / Medium (availability) |

**Key insight**: Because audio is a secondary sensor, SOTIF triggering conditions TC-AUDIO-01 through TC-AUDIO-03 have **low residual risk** — primary sensors (camera, LiDAR, radar) maintain full emergency vehicle detection capability regardless of audio performance limitations. TC-AUDIO-04 is elevated to **medium residual risk** because in-cabin microphone placement inherently attenuates external siren audio by 20-30 dB, significantly reducing the effective detection range; this trade-off is accepted because in-cabin placement enables BIST-based audio path verification and eliminates exterior mounting challenges. TC-AUDIO-05 represents an availability concern (false BIST failures) rather than a safety concern, with medium availability risk managed by retry logic and BIST signal design. The AIQL's scope remains limited to data integrity qualification and BIST-based path verification at the I/O boundary; classifier performance limitations are addressed at the system level by NDAS DRIVE AV multi-sensor fusion architecture.

---

## 13. Traceability Matrix

### 13.1 Safety Goal → Failure Mode → TSR Traceability

| Safety Goal | Failure Mode | TSR(s) | Verification |
|------------|-------------|--------|-------------|
| SG-03 | FM-01: Complete Loss | TSR-AIQL-001, TSR-AIQL-002, TSR-AIQL-007, TSR-AIQL-016 | FI-01 |
| SG-03 | FM-02: Corruption | TSR-AIQL-003, TSR-AIQL-005, TSR-AIQL-007 | FI-03 |
| SG-03 | FM-03: Staleness | TSR-AIQL-001, TSR-AIQL-002, TSR-AIQL-004, TSR-AIQL-007 | FI-01, FI-02 |
| SG-03 | FM-04: Out-of-Range | TSR-AIQL-005, TSR-AIQL-016 | FI-05 |
| SG-03 | FM-05: False Negative | — (system-level: NDAS DRIVE AV multi-sensor fusion) | — (system-level verification) |
| SG-03 | FM-06: False Positive | — (system-level: NDAS DRIVE AV multi-sensor fusion) | — (system-level verification) |
| SG-03 | FM-07: Sequence Disorder | TSR-AIQL-004 | FI-04 |
| SG-03 | FM-08: Common Cause | TSR-AIQL-012, TSR-AIQL-013, TSR-AIQL-007 | FI-07 |
| SG-03 | FM-09: Speaker Failure | TSR-AIQL-015, TSR-AIQL-019, TSR-AIQL-007 | FI-10 |
| SG-03 | FM-10: Coupling Degradation | TSR-AIQL-016, TSR-AIQL-019, TSR-AIQL-007 | FI-11 |
| SG-03 | FM-11: BIST False Failure | TSR-AIQL-018, TSR-AIQL-019 | FI-13 |

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
| TSR-AIQL-015 (BIST Signal Gen) | AoU-011 (Speaker Availability) | BIST requires speaker with specified frequency range and SPL |
| TSR-AIQL-016 (BIST Spectral Match) | AoU-009 (In-Cabin Mic Specs), AoU-012 (Acoustic Coupling) | Spectral match depends on mic placement and acoustic coupling |
| TSR-AIQL-017 (Startup BIST) | AoU-011 (Speaker Availability), AoU-012 (Acoustic Coupling) | Startup BIST requires functional speaker and adequate coupling |
| TSR-AIQL-018 (Periodic BIST) | AoU-011 (Speaker Availability), AoU-012 (Acoustic Coupling) | Periodic BIST requires maintained acoustic path |
| TSR-AIQL-019 (MRM on BIST Failure) | — | AIQL internal behavior; no AoU dependency |

### 13.3 Failure Mode Coverage Analysis

Every failure mode must be addressed by at least one TSR. Every TSR must address at least one failure mode.

| FM | TSR Coverage Count | TSR IDs |
|----|-------------------|---------|
| FM-01 | 4 | TSR-AIQL-001, -002, -007, -016 |
| FM-02 | 3 | TSR-AIQL-003, -005, -007 |
| FM-03 | 4 | TSR-AIQL-001, -002, -004, -007 |
| FM-04 | 2 | TSR-AIQL-005, -016 |
| FM-05 | 0* | — (system-level: NDAS DRIVE AV multi-sensor fusion) |
| FM-06 | 0* | — (system-level: NDAS DRIVE AV multi-sensor fusion) |
| FM-07 | 1 | TSR-AIQL-004 |
| FM-08 | 3 | TSR-AIQL-007, -012, -013 |
| FM-09 | 3 | TSR-AIQL-007, -015, -019 |
| FM-10 | 3 | TSR-AIQL-007, -016, -019 |
| FM-11 | 2 | TSR-AIQL-018, -019 |

*FM-05 and FM-06 are classifier performance limitations (not data integrity failures). As audio is a secondary sensor, these are addressed at the system level by NDAS DRIVE AV multi-sensor fusion with primary sensors (camera, LiDAR, radar) — not by the AIQL.

**Coverage check**: All 9 data integrity and BIST failure modes (FM-01 through FM-04, FM-07 through FM-11) have at least 1 AIQL TSR. FM-05 and FM-06 (classifier performance) are addressed at system level. **PASS**

| TSR | FM Coverage Count | FM IDs |
|-----|-------------------|--------|
| TSR-AIQL-001 | 2 | FM-01, FM-03 |
| TSR-AIQL-002 | 2 | FM-01, FM-03 |
| TSR-AIQL-003 | 1 | FM-02 |
| TSR-AIQL-004 | 2 | FM-03, FM-07 |
| TSR-AIQL-005 | 2 | FM-02, FM-04 |
| TSR-AIQL-007 | 6 | FM-01, FM-02, FM-03, FM-08, FM-09, FM-10 |
| TSR-AIQL-008 | 11 | All (recovery behavior) |
| TSR-AIQL-009 | 0* | Temporal FFI (AIQL self-protection) |
| TSR-AIQL-010 | 0* | Process requirement (all FMs) |
| TSR-AIQL-011 | 0* | Diagnostic (all FMs, observability) |
| TSR-AIQL-012 | 1 | FM-08 |
| TSR-AIQL-013 | 1 | FM-08 |
| TSR-AIQL-015 | 2 | FM-09, FM-10 |
| TSR-AIQL-016 | 3 | FM-01, FM-04, FM-10 |
| TSR-AIQL-017 | 2 | FM-09, FM-10 |
| TSR-AIQL-018 | 2 | FM-10, FM-11 |
| TSR-AIQL-019 | 2 | FM-09, FM-10 |

*TSR-AIQL-009, -010, -011 are cross-cutting requirements (WCET, process, diagnostics) that support the overall safety mechanism integrity rather than addressing specific failure modes.

**Coverage check**: All 17 TSRs address at least 1 failure mode or serve a cross-cutting safety purpose. **PASS**

### 13.4 Verification Coverage

| Verification Item | Failure Modes Covered | TSRs Verified |
|-------------------|----------------------|---------------|
| FI-01 | FM-01, FM-03 | TSR-AIQL-001, -002 |
| FI-02 | FM-03 | TSR-AIQL-001 |
| FI-03 | FM-02 | TSR-AIQL-003 |
| FI-04 | FM-07 | TSR-AIQL-004 |
| FI-05 | FM-04 | TSR-AIQL-005 |
| FI-07 | All (data integrity) | TSR-AIQL-007, -008 |
| FI-08 | FM-08 | TSR-AIQL-012 |
| FI-09 | FM-08 | TSR-AIQL-013 |
| FI-10 | FM-09 | TSR-AIQL-015, -019 |
| FI-11 | FM-10 | TSR-AIQL-016 |
| FI-12 | FM-09, FM-10 | TSR-AIQL-017 |
| FI-13 | FM-10, FM-11 | TSR-AIQL-018 |
| FI-14 | FM-09, FM-10 | TSR-AIQL-019 |

**Coverage check**: All data integrity failure modes (FM-01 through FM-04, FM-07, FM-08) and BIST failure modes (FM-09, FM-10, FM-11) covered by at least one fault injection test. FM-05/FM-06 verified at system level. **PASS**

---

## 14. Open Items

| # | Open Item | Owner | Priority | Target Date | Status |
|---|-----------|-------|----------|-------------|--------|
| OI-01 | **Audio frame interface specification**: Finalize the exact audio frame format, field ordering, and byte alignment with the audio driver development team. Current Section 4.2 is a draft proposal. | Audio SW Lead + Safety Engineering | High | TBD | Open |
| OI-02 | **HARA update for SG-03**: HARA-0003 row added (S2/E3/C2 → ASIL B). Requires formal HARA review board approval to baseline the new safety goal. | Safety Manager | High | TBD | Open |
| OI-03 | **Target compute platform**: Confirm DRIVE AGX Thor (Blackwell) as the target platform for AIQL deployment within Hyperion 10 reference architecture. WCET (TSR-AIQL-009) and MPU/MIG configuration (TSR-AIQL-012) are platform-dependent. Confirm MIG partition allocation for AIQL safety-critical workload. | Platform Architecture | High | TBD | Open |
| OI-04 | **Microphone array specifications**: ~~Confirm microphone placement (exterior vs. cabin)~~ Microphone placement confirmed as **in-cabin** (v0.4). Remaining: finalize count, model selection, and acoustic specifications for in-cabin placement. AoU-009 updated for in-cabin requirements including BIST coupling. Cabin acoustic characterization needed (see OI-09). | Audio HW Lead | Medium | TBD | Partially Resolved |
| OI-05 | **Siren classifier output format**: Confirm the classifier output structure (Section 4.2.3) with the audio ML team. The direction-of-arrival field and confidence metric format need alignment. | Audio ML Team | Medium | TBD | Open |
| OI-06 | **FTTI confirmation**: Validate the 500 ms FTTI through system-level timing analysis including worst-case audio processing latency, NDAS DRIVE AV inference time, and vehicle dynamic response. Current allocation (Section 10.4) is preliminary. | System Safety + Vehicle Dynamics | High | TBD | Open |
| OI-07 | **System-level verification of FM-05/FM-06**: Verify that NDAS DRIVE AV multi-sensor fusion adequately mitigates classifier false negatives (FM-05) and false positives (FM-06) when audio is used as a secondary sensor. This is outside the AIQL scope but must be confirmed at the system level. | System Safety + Perception Team | High | TBD | Open |
| OI-08 | **Speaker specifications for BIST**: Select specific in-cabin speaker(s) for BIST use. Confirm frequency response, SPL output at mic positions, THD, and independent DAC access (AoU-011). Determine if existing entertainment speakers can be shared or if a dedicated BIST speaker is needed. | Audio HW Lead | High | TBD | Open |
| OI-09 | **Cabin acoustic characterization**: Perform acoustic transfer function measurement from BIST speaker(s) to each in-cabin microphone across 500-3000 Hz. Characterize variation across temperature (-40 C to +85 C), HVAC modes, and cabin configurations (doors/windows open/closed, seats occupied/empty). Required to calibrate BIST spectral match reference (TSR-AIQL-016) and validate AoU-012. | Audio HW Lead + Safety Engineering | High | TBD | Open |
| OI-10 | **BIST signal cancellation validation**: Validate that the BIST signal cancellation algorithm (TSR-AIQL-018) achieves >= 30 dB attenuation in representative cabin conditions. Determine if adaptive cancellation is needed or if a fixed reference subtraction is sufficient. | Safety Engineering + Audio SW Lead | Medium | TBD | Open |
| OI-11 | **BIST false failure rate target**: Define quantitative target for false BIST failure rate (preliminary target: < 1 per 1000 driving hours). Perform statistical analysis of expected false failure rate based on cabin noise profiles and BIST signal design. If target is not met, consider noise-adaptive BIST scheduling (OI-14). | Safety Engineering | Medium | TBD | Open |
| OI-12 | **MRM interface specification**: Define the exact interface between AIQL and the vehicle safety manager for MRM requests. Specify signal format, communication channel (dedicated GPIO, safety bus message, shared memory flag), latency requirements, and acknowledgment protocol. | Safety Engineering + Vehicle Platform | High | TBD | Open |
| OI-13 | **Periodic BIST interval optimization**: The 60-second periodic BIST interval is a preliminary value. Optimize based on: (a) expected failure rate dynamics (how fast can acoustic coupling degrade?); (b) cabin audio intrusion tolerance; (c) BIST signal cancellation effectiveness; (d) false failure rate impact. Consider adaptive interval based on vehicle state (longer at highway, shorter in city). | Safety Engineering | Low | TBD | Open |
| OI-14 | **ANC interaction with BIST**: If the vehicle has Active Noise Cancellation (ANC), determine whether ANC will attenuate or distort the BIST signal. If so, define coordination protocol (suspend ANC during BIST window, or account for ANC transfer function in spectral match reference). | Audio HW Lead + Safety Engineering | Medium | TBD | Open |

---

## 15. Appendices

### Appendix A: Glossary

| Term | Definition |
|------|-----------|
| **AIQL** | Audio Input Qualification Layer — the ASIL B safety mechanism defined in this TSC |
| **Alpamayo** | NVIDIA's foundation-model AV research layer — a 10B-parameter Vision Language Action (VLA) model that trains in the cloud (DGX) and produces distilled compact models deployed into the NDAS DRIVE AV on-vehicle stack. Alpamayo sits above NDAS/DRIVE AV; it is not an NDAS component and not part of the DRIVE Thor base platform. |
| **DRIVE AGX Thor** | NVIDIA's centralized AV compute platform based on the Blackwell architecture (2x Thor SoCs, 2000+ FP4 TFLOPS). Hosts DRIVE OS, DriveWorks, and NDAS DRIVE AV. |
| **DRIVE Hyperion 10** | NVIDIA's production-ready AV reference architecture: DRIVE AGX Thor compute + qualified sensor suite (14 cameras, 9 radars, 1 lidar, 12 ultrasonics, 1 exterior mic array, 4 interior cameras). |
| **NDAS (DRIVE AV)** | NVIDIA DRIVE AV Solution — the full-stack autonomous driving software (perception, prediction, mapping, planning, control) running on DRIVE AGX Thor. The AIQL's downstream consumer. |
| **ANC** | Active Noise Cancellation — system that generates anti-phase sound to reduce cabin noise |
| **AoU** | Assumption of Use — interface contract on the QM audio subsystem per ISO 26262-8 Clause 12 |
| **ASIL** | Automotive Safety Integrity Level — risk classification per ISO 26262 (QM, A, B, C, D) |
| **BIST** | Built-In Self-Test — a self-diagnostic mechanism that verifies hardware path integrity using a known test signal |
| **CRC-32** | Cyclic Redundancy Check with 32-bit polynomial — used for data integrity verification |
| **dBFS** | Decibels relative to Full Scale — amplitude measurement where 0 dBFS is the maximum digital level |
| **E2E** | End-to-End protection — communication protection mechanism per AUTOSAR |
| **FFI** | Freedom From Interference — ensures faults in one element cannot corrupt another; analyzed in spatial, temporal, and communication dimensions |
| **FIT** | Failures In Time — failure rate unit (1 FIT = 1 failure per 10^9 hours) |
| **FM** | Failure Mode — a specific way the audio input path can fail |
| **FTTI** | Fault Tolerant Time Interval — maximum time from fault occurrence to safe state |
| **HiL** | Hardware-in-the-Loop — test environment with real target hardware and simulated vehicle |
| **LFM** | Latent Fault Metric — percentage of latent faults covered by safety mechanisms |
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
| Decomposition scheme | Part 9, 5.4.2 | ASIL B = ASIL B(AIQL) + QM(Audio) + QM(Speaker) — documented in Section 8.1 | Complete |
| Independence argument | Part 9, 5.4.3 | Hardware, software, and common cause analysis — documented in Section 8.2 | Complete |
| Sufficient safety mechanisms | Part 9, 5.4.4 | 17 TSRs (16 ASIL B + 1 QM) covering all data integrity and BIST failure modes — documented in Section 7; FM-05/FM-06 addressed at system level | Complete |
| Assumptions of Use | Part 8, 12.4.2 | 12 AoUs on QM audio subsystem — documented in Section 9 | Complete |
| FFI analysis | Part 9, 7 | Spatial, temporal, communication FFI — documented in Section 6.2 | Complete |
| Dependent failure analysis | Part 9, 7 | Common cause analysis — documented in Section 8.2.3 | Complete |
| Verification of decomposition | Part 9, 5.4.5 | Fault injection campaign (149 tests) — documented in Section 11.3 | Planned |

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

### Appendix E: BIST Loopback Architecture

#### E.1 BIST Signal Path

```
+==================+     +============+     +===============+     +=============+
| BIST Signal      |---->| Codec DAC  |---->| In-Cabin      |---->| Acoustic    |
| Generator        |     |            |     | Speaker       |     | Air Path    |
| (within AIQL)    |     | (QM HW)    |     | (QM HW)      |     | (cabin)     |
+==================+     +============+     +===============+     +=============+
                                                                       |
                              +----------------------------------------+
                              |
                              v
+=============+     +============+     +==================+
| In-Cabin    |---->| Codec ADC  |---->| BIST Spectral    |
| Microphone  |     |            |     | Match Analyzer   |
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
| Signal cancellation target | 30 dB | 20-40 dB | Higher = less BIST leakage to NDAS DRIVE AV; harder to achieve |
