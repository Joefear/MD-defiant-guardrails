# Guardrail Decision Format (GDF) v1.0 Standard

**Standard format for AI policy enforcement decisions in regulated environments**

---

## Overview

TThe Guardrail Decision Format (GDF) is a vendor-neutral, regulation-aware standard for representing AI policy enforcement decisions as portable, auditable governance artifacts.

GDF is designed to make AI behavior provable, interoperable, and legally defensible across models, vendors, and deployment environments.
**Status:** Stable  
**Version:** 1.0  
**Specification URI:** https://defiantindustries.com/schemas/gdf/v1.0  
**Schema File:** decision_schema_v1.0_FINAL.json  
**Maintained by:** Defiant Industries Inc.  
**License:** CC0-1.0 (Public Domain)

---

## What Is GDF?

GDF is to AI governance what OpenAPI is to REST APIs:
- **Standardized format** - vendor-neutral, regulation-aware
- **Auditable artifact** - evidence for compliance, not logs
- **Interoperable** - works across AI providers, policy engines, industries

**Use cases:**
- Healthcare (HIPAA minimum necessary)
- Finance (SOX access controls)
- Government (FedRAMP classification levels)
- Legal (attorney-client privilege)

---



## Design Principles

### 1. Auditability First
Every field serves compliance:
- Complete identity chain (USER → ROLE → AGENT → POLICY)
- Explicit authorization vs. denial records
- Human-readable explanations + machine-readable codes
- Tamper-evident integrity fields

### 2. Regulation-Aware
Built-in support for regulatory requirements without being specific to any single regulation.

### 3. Vendor-Neutral
No assumptions about AI model, policy engine, or deployment environment.

### 4. Extensible
Core schema is minimal and universal. Domain-specific features live in `extensions`.

---

## Compatibility Rules

### Semantic Versioning

**Version Format:** MAJOR.MINOR.PATCH

**Compatibility Guarantees:**

**MINOR version increments (1.0 → 1.1):**
- ✅ MAY add optional fields to core schema
- ✅ MAY add new enum values (with backward-compatible defaults)
- ✅ MAY add new extension domains
- ✅ MUST maintain backward compatibility
- ✅ Validators MUST accept v1.0 decisions

**MAJOR version increments (1.x → 2.0):**
- ⚠️ MAY break compatibility
- ⚠️ MAY change required fields
- ⚠️ MAY remove or rename fields
- ⚠️ Requires migration guide

### Unknown Fields

Implementations MUST handle unknown fields gracefully:
- Parsers SHOULD NOT fail on unexpected top-level fields
- `extensions` explicitly allows `additionalProperties: true`
- Forward compatibility: older parsers ignore newer optional fields

### Schema Validation

Decision objects SHOULD validate against the JSON Schema:
```bash
ajv validate -s decision_schema_v1.0_FINAL.json -d my_decision.json
```

---

## Reason Code Registry

Machine-readable codes for `decision_reason_code`, `blocked_data[].reason_code`, and `matched_rules[].reason_code`.

### Standard Codes (Recommended)

**Pattern:** `^[A-Z0-9_]{3,64}$` (alphanumeric + underscore, 3-64 chars)

**Core Decision Codes:**

| Code | Meaning | Typical Decision |
|------|---------|------------------|
| `APPROVED` | Request fully authorized | ALLOW |
| `MINIMUM_NECESSARY_VIOLATION` | Data request exceeds necessary scope | BLOCK |
| `SENSITIVE_CATEGORY_REQUIRES_APPROVAL` | Protected data needs human review | REQUIRE_APPROVAL |
| `WORKFLOW_SCOPE_VIOLATION` | Action not permitted for this workflow | BLOCK |
| `CONFIDENCE_BELOW_THRESHOLD` | AI confidence too low | REQUIRE_APPROVAL |
| `HIGH_RISK_REQUIRES_APPROVAL` | Risk tier triggers approval gate | REQUIRE_APPROVAL |
| `INSUFFICIENT_EVIDENCE` | Cannot verify AI claims | REQUIRE_APPROVAL |
| `UNAUTHORIZED_ACTION` | User role cannot perform action | BLOCK |
| `VIP_PATIENT_REQUIRES_APPROVAL` | Enhanced controls for VIP | REQUIRE_APPROVAL |
| `OFF_LABEL_USE_REQUIRES_APPROVAL` | Medication outside approved indication | REQUIRE_APPROVAL |
| `COST_THRESHOLD_EXCEEDED` | Financial limit reached | REQUIRE_APPROVAL |
| `AFTER_HOURS_ACCESS_BLOCKED` | Access outside permitted hours | BLOCK |
| `IDENTITY_VALIDATION_FAILED` | Authentication insufficient | BLOCK |
| `SESSION_EXPIRED` | Session timeout | BLOCK |

**Domain-Specific Codes:**

Organizations MAY define custom codes following the same pattern.

**Examples:**
- Healthcare: `HIPAA_MIN_NECESSARY_FAIL`, `PSYCHOTHERAPY_NOTES_PROHIBITED`
- Finance: `SOX_SEGREGATION_VIOLATION`, `PCI_CARDHOLDER_DATA_BLOCKED`
- Government: `CLASSIFICATION_INSUFFICIENT`, `CLEARANCE_EXPIRED`

**Best Practice:**
- Use recommended codes when applicable
- Define organization-specific codes in internal registry
- Document custom codes in policy files

---

## Extensions Guidance

The `extensions` object allows domain-specific additions while maintaining core schema compatibility.

### Structure

```json
{
  "extensions": {
    "healthcare": {...},
    "finance": {...},
    "custom_org": {...}
  }
}
```

### Healthcare Extension

**Namespace:** `extensions.healthcare`

**Supported fields:**

**`hallucination_checks` (object):**
```json
{
  "performed": true,
  "unverified_statements": [
    {
      "statement": "Patient allergic to penicillin",
      "source_checked": "allergies.medication_allergies"
    }
  ],
  "contradictions": [
    {
      "statement": "Patient is on insulin",
      "source_ehr_field": "medications.current",
      "actual_value": "Metformin 1000mg BID"
    }
  ]
}
```

**`diagnosis_medication_alignment` (object):**
```json
{
  "diagnosis_code": "E11.9",
  "medication": "Ozempic",
  "clinically_appropriate": true,
  "contraindications": []
}
```

### Future Extensions

**Planned:**
- `extensions.finance` - SOX controls, transaction limits, audit trails
- `extensions.government` - Classification levels, clearance checks, need-to-know
- `extensions.legal` - Attorney-client privilege, ethical walls, conflict checks

**Proposal Process:**
1. Open issue in GDF spec repository
2. Describe use case and proposed fields
3. Show example JSON
4. Community review → merge into spec

---

## Observability Integration

GDF supports distributed tracing via W3C Trace Context:

### Correlation IDs

**`request_id` (string):**  
Upstream request identifier from calling application.

**`trace_id` (string, pattern `^[0-9a-f]{32}$`):**  
W3C Trace Context trace ID for end-to-end observability.

**`span_id` (string, pattern `^[0-9a-f]{16}$`):**  
W3C Trace Context span ID (optional).

### Integration Example

```python
# OpenTelemetry integration
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("guardrail_decision") as span:
    decision = guardrail.evaluate(policy, request)
    
    # Add trace context to GDF decision
    decision["trace_id"] = format(span.get_span_context().trace_id, '032x')
    decision["span_id"] = format(span.get_span_context().span_id, '016x')
```

---

## Field Reference

### Required Fields

All GDF decisions MUST include:
- `decision_id` - Unique UUID v4
- `decision` - ALLOW | REQUIRE_APPROVAL | BLOCK
- `timestamp` - ISO 8601 with timezone
- `policy_id` - Policy identifier
- `policy_version` - Version for reproducibility
- `workflow_type` - What workflow this applies to
- `identity_chain` - Complete audit trail
- `requested_data` - PHI/data requested (may be empty array)
- `requested_actions` - Actions requested (may be empty array)
- `risk_assessment` - Risk evaluation with `risk_tier`

### Optional But Recommended

**For Production Deployments:**
- `decision_reason_code` - Machine-readable reason
- `policy_hash` - Tamper-evident policy verification
- `authorized_data` / `blocked_data` - What was actually allowed/denied
- `authorized_actions` / `blocked_actions` - Action enforcement
- `notifications` - Proof of compliance officer alerts
- `latency_ms` - Performance monitoring

**For Enterprise Integration:**
- `request_id` / `trace_id` / `span_id` - Observability
- `integrity` - Digital signatures, hash chains
- `audit_metadata` - SIEM correlation, retention

**For Complex Decisions:**
- `matched_rules` - Which rules triggered
- `human_gate` - Approval requirements
- `evidence_refs` - AI claim verification

---

## Decision Lifecycle

### 1. Request Arrives
```
User → AI Agent → Policy Engine (Guardrail)
```

### 2. Policy Evaluation
```
Load policy + version
Validate identity chain
Check data access rules
Check action permissions
Calculate risk tier
Determine confidence (if applicable)
```

### 3. Decision Generated
```
Create decision object (GDF v1.0)
Sign decision (if integrity enabled)
Log to audit trail
Send notifications (if violations)
```

### 4. Decision Enforced
```
If ALLOW: Release authorized_data, allow authorized_actions
If BLOCK: Deny all, notify compliance
If REQUIRE_APPROVAL: Route to reviewer, wait for outcome
```

### 5. Compliance Export
```
Aggregate decisions for time period
Export to CSV/JSON
Provide to auditors
Archive per retention policy
```

---

## Integration Patterns

### SIEM Integration

**Splunk:**
```
sourcetype=gdf:decision
| stats count by decision_reason_code, risk_tier
| where decision="BLOCK"
```

**QRadar:**
```sql
SELECT decision_reason_code, COUNT(*) 
FROM gdf_decisions 
WHERE decision = 'BLOCK' 
GROUP BY decision_reason_code
```

### Analytics

**Python:**
```python
import json

# Load decisions
with open('decisions.jsonl') as f:
    decisions = [json.loads(line) for line in f]

# Analyze violations
blocked = [d for d in decisions if d['decision'] == 'BLOCK']
by_reason = {}
for d in blocked:
    code = d.get('decision_reason_code', 'UNKNOWN')
    by_reason[code] = by_reason.get(code, 0) + 1

print(f"Top violation reasons: {sorted(by_reason.items(), key=lambda x: x[1], reverse=True)[:5]}")
```

### Compliance Reporting

**Export for OCR Audit:**
```bash
# Filter decisions for date range
jq 'select(.timestamp >= "2026-01-01" and .timestamp < "2026-02-01")' decisions.jsonl > jan_2026.jsonl

# Convert to CSV
jq -r '[.decision_id, .timestamp, .decision, .decision_reason_code, .risk_assessment.risk_tier, .latency_ms] | @csv' jan_2026.jsonl > jan_2026.csv
```

---

## Validation Examples

### Valid Decision (Minimal)

```json
{
  "decision_id": "550e8400-e29b-41d4-a716-446655440000",
  "decision": "ALLOW",
  "timestamp": "2026-01-18T14:23:45.123Z",
  "policy_id": "policy_prior_auth_v1",
  "policy_version": "1.0",
  "workflow_type": "prior_authorization",
  "identity_chain": {
    "user_id": "NPI_1234567890",
    "user_role": "attending_physician",
    "agent_id": "claude-instance-42"
  },
  "requested_data": [],
  "requested_actions": [
    {"action": "draft_prior_authorization"}
  ],
  "risk_assessment": {
    "risk_tier": "MEDIUM"
  }
}
```

### Invalid Decisions

**Missing required field:**
```json
{
  "decision_id": "...",
  "decision": "ALLOW"
  // Missing: timestamp, policy_id, policy_version, ...
}
```
❌ Fails validation

**Invalid enum:**
```json
{
  "decision": "MAYBE"  // Not in enum
}
```
❌ Fails validation

**Invalid reason code pattern:**
```json
{
  "decision_reason_code": "approved"  // Lowercase, fails ^[A-Z0-9_]{3,64}$
}
```
❌ Fails validation

---

## Reference Implementation

**Defiant Guardrail** is the reference implementation of GDF v1.0:
- Policy engine that emits GDF decisions
- Healthcare starter pack with production examples
- Validates all decisions against schema

**But GDF is not tied to Guardrail.**

Any policy engine can emit GDF decisions:
- Custom in-house governance systems
- Commercial AI security platforms
- Cloud provider policy services

---

## Contributing

GDF is designed to be community-driven.

**To propose changes:**
1. Open issue: github.com/defiant-industries/gdf-spec/issues
2. Describe use case and proposed field/change
3. Show example JSON demonstrating need
4. Community review → RFC → merge

**Governance:**
- Defiant Industries maintains spec
- Community input via issues/PRs
- Semantic versioning commitment
- Backward compatibility in MINOR versions

---

## Resources

**Specification URI:** https://defiantindustries.com/schemas/gdf/v1.0  
**JSON Schema:** decision_schema_v1.0_FINAL.json  
**Examples:** ../../examples/decisions/  
**Reference Implementation:** Defiant Guardrail  
**Discussion Forum:** [To be established]  
**Issue Tracker:** github.com/defiant-industries/gdf-spec/issues

---

## Contact

**Specification Questions:** gdf-spec@defiantindustries.com  
**Implementation Support:** support@defiantindustries.com  
**Adoption Discussion:** partnerships@defiantindustries.com  

---

**Version:** 1.0  
**Last Updated:** January 18, 2026  
**Status:** Stable  
**License:** CC0-1.0 (Public Domain for schema, reference implementation separately licensed)

---

**Adopt GDF. Make AI governance interoperable.**
