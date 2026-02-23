# Safety Analysis Templates for Autonomous Vehicle Development

ISO 26262 and ISO 21448 (SOTIF) safety analysis CSV templates with realistic autonomous vehicle example data.

## Purpose

These templates provide structured starting points for safety analysis activities across the V-model development lifecycle. Each CSV is formatted for direct import into Microsoft Excel while maintaining traceability between analysis levels.

## Folder Structure

```
safety-analysis-templates/
  README.md
  reference/                    # Lookup tables and classification scales
    REF-001_ASIL_Determination_Matrix.csv
    REF-002_Severity_Exposure_Controllability_Scales.csv
    REF-003_Failure_Categories_Legend.csv
  vehicle-level/                # Hazard analysis at item level
    HARA-001_Hazard_Analysis_Risk_Assessment.csv
  system-level/                 # System-level failure and safety analyses
    FMEA-001_System_FMEA.csv
    FTA-001_Fault_Tree_Analysis.csv
    SOTIF-001_SOTIF_Analysis.csv
    TSC-001_Audio_Input_Qualification_Layer.md
  component-level/              # Hardware component failure analysis
    FMEDA-001_Hardware_FMEDA.csv
```

## Template Descriptions

| File | Standard | Description |
|------|----------|-------------|
| **REF-001** | ISO 26262-3 | ASIL determination matrix (S1-S3 x E1-E4 x C1-C3) |
| **REF-002** | ISO 26262-3 | Severity, Exposure, and Controllability classification scales |
| **REF-003** | ISO 26262-5 | Failure categories (Safe, SPF, RF, MPF-D, MPF-P, MPF-L) with SPFM/LFM thresholds |
| **HARA-001** | ISO 26262-3 | Hazard Analysis and Risk Assessment with safety goals and FTTI |
| **FMEA-001** | ISO 26262-5 | System-level Failure Modes and Effects Analysis |
| **FTA-001** | ISO 26262-5/9 | Fault Tree Analysis with gate logic and failure rates |
| **SOTIF-001** | ISO 21448 | SOTIF analysis for triggering conditions and functional insufficiencies |
| **TSC-001** | ISO 26262-4/8/9 | Technical Safety Concept — Audio Input Qualification Layer (AIQL) for Alpamayo boundary qualification |
| **FMEDA-001** | ISO 26262-5 | Hardware Failure Modes, Effects, and Diagnostic Analysis |

## Traceability Flow

The templates support end-to-end traceability through the safety lifecycle:

```
HARA-001 (Safety Goals)
    |
    +---> TSC-001 (Technical Safety Concept qualifying QM inputs at ASIL boundary)
    |       |
    |       +---> Failure modes, TSRs, AoUs, ASIL decomposition, fault injection
    |
    v
FMEA-001 (Failure modes traced to safety goal violations)
    |
    +---> FTA-001 (Fault trees decomposing top events from FMEA)
    |
    +---> SOTIF-001 (Performance limitations traced to safety goals)
    |
    v
FMEDA-001 (Hardware failure rates supporting FMEA and FTA)
```

- **HARA safety goals** (e.g., SG-01, SG-02, SG-03) are referenced in FMEA vehicle effects, SOTIF hazardous behaviors, and TSC safety requirements
- **TSC documents** define safety mechanisms (technical safety requirements) to qualify QM inputs at ASIL boundaries, traced bidirectionally to HARA safety goals and FMEA/SOTIF failure modes
- **FMEA failure modes** inform FTA top events and basic event identification
- **FTA failure rates** use component-level data from FMEDA
- **SOTIF scenarios** complement FMEA by addressing performance limitations rather than hardware faults

## ASIL Quick Reference

| ASIL | Integrity Level | SPFM Target | LFM Target |
|------|----------------|-------------|------------|
| QM   | Quality Management (no safety requirement) | N/A | N/A |
| A    | Lowest safety integrity | N/A | N/A |
| B    | Low safety integrity | >= 90% | >= 60% |
| C    | Medium safety integrity | >= 97% | >= 80% |
| D    | Highest safety integrity | >= 99% | >= 90% |

## Excel Usage Notes

- **Opening CSVs**: In Excel, use File > Open and select the CSV. All files use UTF-8 encoding with comma delimiters.
- **ID Formatting**: All IDs use alphanumeric prefixes (e.g., `HARA-0001`) to prevent Excel from stripping leading zeros or converting to numbers.
- **Fields with commas**: Fields containing commas are enclosed in double quotes and will parse correctly in Excel.
- **Preserving format**: When saving from Excel, use "CSV UTF-8 (Comma delimited)" to maintain encoding.

## Naming Convention

```
[TYPE]-[NNN]_[Description].csv

TYPE:   REF   = Reference / lookup table
        HARA  = Hazard Analysis and Risk Assessment
        FMEA  = Failure Modes and Effects Analysis
        FTA   = Fault Tree Analysis
        SOTIF = SOTIF Analysis
        TSC   = Technical Safety Concept
        FMEDA = Failure Modes Effects and Diagnostic Analysis
NNN:    Sequential number (001, 002, ...)
```

## Standard References

- **ISO 26262:2018** — Road vehicles — Functional safety (Parts 1-12)
- **ISO 21448:2022** — Road vehicles — Safety of the intended functionality (SOTIF)
- **IEC 61508** — Functional safety of electrical/electronic/programmable electronic safety-related systems
- **SN 29500** — Failure rates of electronic components (Siemens standard)
