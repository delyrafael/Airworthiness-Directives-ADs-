# Airworthiness-Directives-ADs

# Airworthiness Directive Applicability System

## Overview

This system extracts compliance rules from regulatory Airworthiness Directives (ADs) and provides programmatic evaluation of whether specific aircraft configurations are affected.

## Schema Design

### Core Data Model

The system uses three main data structures:

#### 1. `ApplicabilityRules`
The primary container for AD compliance rules:

```python
{
  "ad_id": str,                    # Unique identifier (e.g., "FAA-2025-23-53")
  "authority": str,                # Issuing authority (FAA/EASA)
  "issue_date": str,               # ISO date
  "effective_date": str,           # ISO date
  "aircraft_models": List[str],    # All affected model designations
  "msn_constraints": {             # Optional MSN filters
    "included_msns": List[str],
    "excluded_msns": List[str],
    "msn_ranges": List[{"min": int, "max": int}]
  },
  "modification_constraints": [    # Mod/SB that affect applicability
    {
      "mod_number": str,
      "service_bulletin": str,
      "production_embodiment": bool,
      "excludes_if_installed": bool,
      "requires_if_missing": bool
    }
  ],
  "additional_notes": List[str]
}
```

#### 2. `AircraftConfiguration`
Represents a specific aircraft instance:

```python
{
  "model": str,              # e.g., "A320-214"
  "msn": str,                # Manufacturer Serial Number
  "modifications": List[str] # Applied mods/SBs
}
```

#### 3. `ApplicabilityResult`
Evaluation outcome:
- `APPLICABLE` - Aircraft is affected by the AD
- `NOT_APPLICABLE` - Aircraft is not affected
- `REQUIRES_INSPECTION` - Cannot determine without inspection (future use)

## Extraction Approach

### 1. Document Analysis
For each AD, I extracted:
- **Header Information**: AD number, authority, dates
- **Applicability Section**: Which aircraft models are affected
- **MSN Constraints**: Specific serial numbers or ranges
- **Modification History**: Which mods exclude or include aircraft

### 2. FAA AD 2025-23-53 (MD-11/DC-10 Engine Pylon)

**Key Findings:**
- Emergency AD issued after engine pylon detachment accident
- Applies to ALL MD-11, MD-11F, MD-10, and DC-10 variants
- No MSN restrictions - every aircraft of these types is affected
- No modification exclusions - the structural design is the issue
- Prohibits flight until inspection completed

**Extraction Logic:**
```
IF aircraft.model IN [MD-11, MD-11F, MD-10-*, DC-10-*]
  THEN APPLICABLE
ELSE NOT_APPLICABLE
```

### 3. EASA AD 2025-0254 (A320 Landing Gear Actuator)

**Key Findings:**
- Affects specific A320 and A321 variants
- Several production modifications exclude aircraft:
  - mod 24591 (mentioned in assignment)
  - mod 24977 (mentioned in assignment)
- Service Bulletins that terminate applicability:
  - A320-57-1089 (mentioned in assignment - Rev 04)
  - A320-57-1060, -1088, -1101, -1126, -1256

**Extraction Logic:**
```
IF aircraft.model IN [A320-211 through A320-233, A321-111/112/131]
  AND NOT has_modification(24591, 24977)
  AND NOT has_service_bulletin(1089, 1060, 1088, 1101, 1126, 1256)
  THEN APPLICABLE
ELSE NOT_APPLICABLE
```

## Design Decisions

### 1. Flexible Modification Matching
Modifications can be specified multiple ways:
- "mod 24591"
- "24591 (production)"
- "modification 24591"

The system uses substring matching to handle variants.

### 2. Model Name Normalization
Aircraft models are normalized (remove hyphens, spaces, uppercase) to match variants:
- "A320-214" matches "A320214" and "A320"
- Handles both specific variants and family-level matches

### 3. Separation of Concerns
- `ApplicabilityRules`: Storage format (database-ready)
- `ADApplicabilityEvaluator`: Business logic
- `AircraftConfiguration`: Input data model

### 4. Explicit Constraints
Rather than embedding logic in code, constraints are data-driven:
- `excludes_if_installed`: Having this mod makes AD not applicable
- `requires_if_missing`: Not having this mod makes AD applicable
- `production_embodiment`: Mod applied during manufacturing

## Test Results

| Model | MSN | Modifications | FAA 2025-23-53 | EASA 2025-0254 |
|-------|-----|---------------|----------------|----------------|
| MD-11 | 48123 | None | **YES** | NO |
| DC-10-30F | 47890 | None | **YES** | NO |
| Boeing 737-800 | 30123 | None | NO | NO |
| A320-214 | 5234 | None | NO | **YES** |
| A320-232 | 6789 | mod 24591 (production) | NO | NO |
| A320-214 | 7456 | SB A320-57-1089 Rev 04 | NO | NO |
| A321-111 | 8123 | None | NO | **YES** |
| A321-112 | 364 | mod 24977 (production) | NO | NO |
| A319-100 | 9234 | None | NO | NO |
| MD-10-10F | 46234 | None | **YES** | NO |

### Result Analysis

**FAA AD 2025-23-53 (Engine Pylon):**
- Affects all 3 MD/DC-10 aircraft: MD-11 (48123), DC-10-30F (47890), MD-10-10F (46234)
- Does not affect Boeing 737, any Airbus A320 family

**EASA AD 2025-0254 (Landing Gear):**
- Affects 2 aircraft without excluding modifications: A320-214 (5234), A321-111 (8123)
- Does not affect:
  - A320-232 with mod 24591
  - A320-214 with SB A320-57-1089
  - A321-112 with mod 24977
  - A319-100 (not in affected models list)
  - Any MD/DC-10 aircraft

## Usage Examples

### Loading Rules from JSON
```python
evaluator = ADApplicabilityEvaluator()
evaluator.load_rules(AD_RULES_DATA)
```

### Evaluating an Aircraft
```python
aircraft = AircraftConfiguration(
    model="A320-214",
    msn="5234",
    modifications=[]
)

result, reason = evaluator.evaluate(aircraft, "EASA-2025-0254")
print(f"{result.value}: {reason}")
# Output: applicable: Aircraft matches all applicability criteria
```

### Checking with Modifications
```python
aircraft = AircraftConfiguration(
    model="A320-214",
    msn="7456",
    modifications=["SB A320-57-1089 Rev 04"]
)

result, reason = evaluator.evaluate(aircraft, "EASA-2025-0254")
# Output: not_applicable: Aircraft has excluding modification: A320-57-1089
```

## Scalability Considerations

### Database Storage
The JSON schema maps directly to relational tables:

```sql
CREATE TABLE airworthiness_directives (
    ad_id VARCHAR(50) PRIMARY KEY,
    authority VARCHAR(10),
    issue_date DATE,
    effective_date DATE
);

CREATE TABLE ad_affected_models (
    ad_id VARCHAR(50) REFERENCES airworthiness_directives,
    model VARCHAR(50),
    PRIMARY KEY (ad_id, model)
);

CREATE TABLE ad_modification_constraints (
    id SERIAL PRIMARY KEY,
    ad_id VARCHAR(50) REFERENCES airworthiness_directives,
    mod_number VARCHAR(50),
    service_bulletin VARCHAR(50),
    excludes_if_installed BOOLEAN,
    production_embodiment BOOLEAN
);
```

### Performance Optimization
For production systems with thousands of aircraft:
1. Index aircraft models and MSNs
2. Cache compiled rules
3. Pre-compute applicability for static configurations
4. Use batch evaluation for fleet-wide queries

## Limitations and Future Work

### Current Limitations
1. **Manual Extraction**: Rules are currently extracted manually from AD text
2. **No Effective Date Logic**: System doesn't check if AD is currently in effect
3. **No Supersession Handling**: Doesn't automatically apply superseding ADs
4. **Limited Temporal Logic**: Can't handle "after X flight hours" conditions

### Future Enhancements
1. **LLM-Based Extraction**: Use GPT-4/Claude to automatically extract rules from PDF ADs
2. **Flight Hours/Cycles**: Add temporal constraints (hours, cycles, calendar time)
3. **Manufacturing Date Constraints**: Support date-based applicability
4. **Compliance Tracking**: Track which aircraft have completed required actions
5. **Revision History**: Track AD revisions and their impact on applicability

## Tools Used
- **Web Search**: To find official AD documents and verify details
- **PDF Analysis**: Reading Federal Register and EASA AD publications
- **Domain Knowledge**: Understanding aviation compliance terminology (MSN, SB, modifications)

## Extraction Process
1. Fetched official AD sources from FAA.gov and ad.easa.europa.eu
2. Identified applicability sections in regulatory text
3. Extracted model lists, MSN ranges, and modification constraints
4. Cross-referenced with Service Bulletin information
5. Validated against test cases in assignment

## Confidence Assessment
- **FAA AD 2025-23-53**: High confidence (straightforward model-based applicability)
- **EASA AD 2025-0254**: High confidence (clear mod exclusions documented)

Both ADs have clear applicability criteria without ambiguous conditions.
