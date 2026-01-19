# Guardrail Decision Format (GDF) v1.0

**Standard format for AI policy enforcement decisions in regulated environments**

---

## What Is GDF?

The Guardrail Decision Format (GDF) is a standardized JSON schema for documenting policy enforcement decisions made by AI governance systems.

**Think of it as:**
- OpenAPI for AI governance
- A "decision receipt" proving compliance
- The audit artifact compliance teams rely on

**Not specific to:**
- Any particular AI model or vendor
- Healthcare (works for finance, government, legal, etc.)
- A single implementation (reference implementation: Defiant Guardrail)

---

## Why GDF Exists

**The Problem:**  
AI systems in regulated environments need to prove:
1. What data was requested
2. What was authorized vs. blocked
3. Why decisions were made  
4. Who made the request
5. What policy governed it

Without a standard format, every vendor invents their own logging structure. This makes:
- Audits harder (inconsistent evidence)
- Tool integration brittle (custom parsers for each vendor)
- Compliance expensive (manual review of vendor-specific logs)

**The Solution:**  
GDF provides a vendor-neutral, regulation-aware decision format that works across:
- AI providers (Claude, ChatGPT, custom models)
- Regulated industries (healthcare, finance, government, legal)
- Deployment models (cloud, on-prem, hybrid)

---

## Core Design Principles

### 1. Auditability First
Every field serves compliance:
- Complete identity chain (USER → ROLE → AGENT → POLICY)
- Explicit authorization vs. denial records
- Human-readable explanations + machine-readable codes
- Tamper-evident integrity fields

### 2. Regulation-Aware
Built-in support for:
- HIPAA (minimum necessary, audit controls)
- GDPR (data minimization, purpose limitation)
- SOX (access controls, audit trails)
- FedRAMP (identity chain, encryption)

### 3. Vendor-Neutral
No assumptions about:
- Which AI model is running
- Which policy engine enforces rules
- Which systems integrate
- Which industry deploys it

### 4. Extensible
Core schema is minimal; domain extensions live in `extensions`:
- `extensions.healthcare` - Clinical checks, diagnosis alignment
- `extensions.finance` - Transaction limits, SOX controls
- `extensions.government` - Classification levels, clearance checks

---

## Schema Overview

### Required Fields (Core)
```json
{
  "decision_id": "UUID",
  "decision": "ALLOW | REQUIRE_APPROVAL | BLOCK",
  "decision_reason_code": "APPROVED | MINIMUM_NECESSARY_VIOLATION | ...",
  "timestamp": "ISO 8601",
  "policy_id": "string",
  "policy_version": "string",
  "workflow_type": "string",
  "identity_chain": {...},
  "requested_data": [...],
  "requested_actions": [...],
  "risk_assessment": {...}
}
```

### Optional Fields (Recommended)
```json
{
  "authorized_data": [...],
  "blocked_data": [...],
  "authorized_actions": [...],
  "blocked_actions": [...],
  "human_gate": {...},
  "notifications": [...],
  "integrity": {...}
}
```

---

## Example: Healthcare Prior Authorization

**Scenario:** AI agent requests complete medical history for prior auth.  
**Decision:** BLOCK (exceeds minimum necessary)

```json
{
  "decision_id": "550e8400-e29b-41d4-a716-446655440001",
  "decision": "BLOCK",
  "decision_reason_code": "MINIMUM_NECESSARY_VIOLATION",
  "timestamp": "2026-01-18T14:23:45.123Z",
  
  "policy_id": "policy_prior_authorization_v1",
  "policy_version": "1.0",
  
  "identity_chain": {
    "user_id": "NPI_1234567890",
    "user_role": "attending_physician",
    "agent_id": "claude-sonnet-4-instance-42"
  },
  
  "requested_data": [
    {"field": "complete_medical_history"}
  ],
  
  "blocked_data": [
    {
      "field": "complete_medical_history",
      "reason": "Exceeds HIPAA minimum necessary for prior authorization",
      "reason_code": "MINIMUM_NECESSARY_VIOLATION",
      "alternative": "Request specific clinical fields (diagnosis, medications, allergies)"
    }
  ],
  
  "risk_assessment": {
    "risk_tier": "HIGH",
    "confidence_applicable": false
  },
  
  "notifications": [
    {
      "recipient_role": "compliance_officer",
      "channel": "EMAIL",
      "sent_at": "2026-01-18T14:23:46.000Z"
    }
  ],
  
  "latency_ms": 42
}
```

**Compliance value:**
- OCR auditor sees: "What was requested? What was blocked? Why? Who was notified?"
- SIEM sees: `decision_reason_code: MINIMUM_NECESSARY_VIOLATION`
- Policy team sees: `policy_id + policy_version` for reproducibility

---

## Adoption

### Who Should Use GDF?

**AI Governance Vendors:**
- Emit GDF decisions from your policy engine
- Become compatible with GDF-aware tools

**Enterprise Buyers:**
- Require GDF support from AI governance vendors
- Build analytics on standardized decision format

**Auditors/Compliance:**
- Request GDF exports for audit reviews
- Build compliance reporting tools against GDF schema

### Reference Implementation

**Defiant Guardrail** is the reference implementation of GDF:
- Policy engine that emits GDF v1.0 decisions
- Healthcare starter pack demonstrates GDF in production
- Open-source examples show real GDF outputs

**But GDF is not tied to Guardrail.**  
Any policy engine can emit GDF decisions.

---

## Versioning

**Current Version:** 1.0  
**Backward Compatibility:** Semantic versioning (semver)
- Minor version changes (1.1, 1.2): Add optional fields, maintain backward compatibility
- Major version changes (2.0): May break compatibility

**Upgrade Path:**  
- Version 1.x decisions validate against GDF 1.0 schema
- Tools should handle unknown fields gracefully (`additionalProperties: true` in extensions)

---

## Schema Details

### Field Naming Conventions

**Enums:** UPPERCASE  
```json
"decision": "ALLOW | REQUIRE_APPROVAL | BLOCK"
"risk_tier": "LOW | MEDIUM | HIGH | CRITICAL"
```

**Timestamps:** ISO 8601 with timezone  
```json
"timestamp": "2026-01-18T14:23:45.123Z"
```

**IDs:** Descriptive prefix + value  
```json
"decision_id": "550e8400-..."
"policy_id": "policy_prior_authorization_v1"
```

### Machine-Readable Reason Codes

Every decision includes `decision_reason_code` for analytics:

| Code | Meaning |
|------|---------|
| `APPROVED` | Request fully authorized |
| `MINIMUM_NECESSARY_VIOLATION` | Data request exceeds necessary scope |
| `SENSITIVE_CATEGORY_REQUIRES_APPROVAL` | Protected data needs human review |
| `WORKFLOW_SCOPE_VIOLATION` | Action not permitted for this workflow |
| `CONFIDENCE_BELOW_THRESHOLD` | AI confidence too low, needs review |
| `HIGH_RISK_REQUIRES_APPROVAL` | Risk tier triggers approval gate |
| `UNAUTHORIZED_ACTION` | User role cannot perform action |
| `COST_THRESHOLD_EXCEEDED` | Financial limit reached |

**Extensibility:** Organizations can add custom codes in extensions.

### Identity Chain

Complete audit trail of who made the request:

```json
"identity_chain": {
  "user_id": "NPI_1234567890",           // Human who initiated
  "user_role": "attending_physician",    // Their role
  "agent_id": "claude-instance-42",      // AI agent
  "session_id": "sess_abc123",           // Session context
  "system_of_origin": "epic_smartfhir",  // Where it came from
  "idp": "okta",                         // Identity provider
  "auth_strength": "MFA"                 // How they authenticated
}
```

**Why this matters:**  
USER → ROLE → AGENT → POLICY creates non-repudiable audit trail.

### Confidence Scoring

For decisions involving AI claims verification:

```json
"risk_assessment": {
  "confidence_score": 0.92,
  "confidence_method": "evidence_coverage",
  "confidence_applicable": true
}
```

**When `confidence_applicable: false`:**  
Policy-based blocks (e.g., "complete history prohibited") don't need confidence.

**When `confidence_score: null`:**  
Confidence couldn't be calculated but is applicable.

### Tamper Evidence

Optional integrity fields for append-only logs:

```json
"integrity": {
  "hash_chain_prev": "e3b0c44...",
  "digital_signature": "a1b2c3d...",
  "signature_algorithm": "ed25519",
  "signed_fields": ["decision_id", "timestamp", "decision"]
}
```

---

## Extensions

### Core vs. Extensions

**Core schema:** Universal fields that apply to all AI governance decisions  
**Extensions:** Domain-specific additions

### Healthcare Extension

```json
"extensions": {
  "healthcare": {
    "hallucination_checks": [...],
    "diagnosis_medication_alignment": {...}
  }
}
```

### Future Extensions

- `extensions.finance` - SOX controls, transaction limits
- `extensions.government` - Classification levels, clearance checks
- `extensions.legal` - Attorney-client privilege, ethical walls

---

## Compliance Mapping

### HIPAA
- **Minimum Necessary (164.502(b))**: `requested_data` vs `authorized_data`
- **Audit Controls (164.312(b))**: Complete GDF decision object
- **Access Control (164.312(a))**: `identity_chain`
- **Integrity (164.312(c))**: `integrity` section

### GDPR
- **Data Minimization (Article 5)**: `blocked_data` enforcement
- **Purpose Limitation (Article 5)**: `purpose` field
- **Right to Explanation (Article 22)**: `decision_explanation_human_readable`

### SOX
- **IT General Controls**: `identity_chain` + `policy_version`
- **Audit Trail**: `audit_metadata`
- **Change Management**: `policy_hash` for verification

---

## Tools & Integrations

### Validation
```bash
# Validate decision against GDF schema
ajv validate -s decision_schema.json -d my_decision.json
```

### Analytics
```python
import gdf

# Parse GDF decision
decision = gdf.load("decision.json")

# Query decisions
blocked_phi = [d for d in decisions if d.decision == "BLOCK"]

# Generate compliance report
report = gdf.compliance_report(decisions, regulation="HIPAA")
```

### SIEM Integration
```
# Splunk
sourcetype=gdf:decision
| stats count by decision_reason_code

# QRadar
SELECT decision_reason_code, COUNT(*) 
FROM gdf_decisions 
WHERE decision = 'BLOCK'
GROUP BY decision_reason_code
```

---

## Contributing

GDF is designed to be community-driven.

**To propose changes:**
1. Open issue at [GitHub repository]
2. Describe use case and proposed field
3. Show example JSON
4. Community review → merge into next version

**Governance:**
- Semantic versioning
- Backward compatibility in minor versions
- Major versions require RFC process

---

## License

GDF schema is open and freely adoptable.  
Reference implementation (Defiant Guardrail) has separate license.

---

## Resources

**Schema:** `decision_schema.json` (this folder)  
**Examples:** `../../examples/` (healthcare prior auth)  
**Reference Implementation:** Defiant Guardrail  
**Specification URI:** https://defiantindustries.com/schemas/gdf/v1.0  

---

## Contact

**Schema Questions:** gdf-spec@defiantindustries.com  
**Implementation Support:** support@defiantindustries.com  
**Adoption Discussion:** partnerships@defiantindustries.com  

---

**Version:** 1.0  
**Last Updated:** January 18, 2026  
**Status:** Stable

**Adopt GDF. Make AI governance interoperable.**