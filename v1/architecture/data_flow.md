# Guardrail Data Flow

**Detailed Request/Response Flow for Policy Enforcement**

---

## Overview

This document traces a single AI request through Guardrail's policy enforcement pipeline, showing exactly what happens at each step, what data is evaluated, and what decisions are made.

---

## Example Scenario: Prior Authorization Request

**Context:**
- Dr. Smith (attending physician) is using Epic with Claude integration
- Patient is 52-year-old with Type 2 Diabetes
- Physician wants to check insurance coverage for Ozempic (diabetes medication)

---

## Flow Diagram

```
┌─────────────┐
│   START     │
│ User Action │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 1: User Initiates Request                      │
│                                                      │
│ Dr. Smith in Epic EHR:                               │
│ "Check coverage for Ozempic for this patient"       │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 2: AI Agent Receives Request                   │
│                                                      │
│ Claude Healthcare Instance:                          │
│ - Parses natural language intent                    │
│ - Determines need to access PHI                     │
│ - Formulates data/action requests                   │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 3: Request Intercepted by Guardrail            │
│                                                      │
│ Guardrail receives structured request:              │
│ {                                                    │
│   "user_id": "NPI_1234567890",                       │
│   "user_role": "attending_physician",                │
│   "ai_agent_id": "claude-sonnet-4-instance-42",      │
│   "workflow_type": "prior_authorization",            │
│   "requested_data": [                                │
│     "patient_demographics.age",                      │
│     "diagnoses.primary_icd10",                       │
│     "medications.current_list",                      │
│     "allergies.medication_allergies"                 │
│   ],                                                 │
│   "requested_actions": [                             │
│     "draft_prior_authorization_request"              │
│   ]                                                  │
│ }                                                    │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 4: Identity Chain Validation                   │
│                                                      │
│ Guardrail verifies:                                  │
│ ✓ User NPI_1234567890 authenticated via Okta        │
│ ✓ Role "attending_physician" matches AD groups      │
│ ✓ AI agent "claude-instance-42" registered          │
│ ✓ Request originated from Epic FHIR endpoint        │
│                                                      │
│ Identity Chain Built:                                │
│ USER:NPI_1234567890 →                                │
│ ROLE:attending_physician →                           │
│ AGENT:claude-instance-42 →                           │
│ POLICY:policy_prior_authorization_v1                 │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 5: Policy Loading                              │
│                                                      │
│ Guardrail loads:                                     │
│ - policy_prior_authorization_v1.yaml                 │
│ - phi_tags.yaml (PHI classification)                 │
│ - action_catalog.yaml (action definitions)           │
│                                                      │
│ Policy identifies:                                   │
│ - Permitted data for prior auth workflow             │
│ - Prohibited data (complete history, mental health)  │
│ - Allowed actions (draft) vs blocked (submit)        │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 6: PHI Access Evaluation                       │
│                                                      │
│ For each requested data field:                       │
│                                                      │
│ 1. "patient_demographics.age"                        │
│    ✓ Check: permitted_data.patient_demographics.age  │
│    ✓ Justification: Age-based formulary requirements │
│    ✓ Risk level: LOW                                 │
│    → ALLOW                                           │
│                                                      │
│ 2. "diagnoses.primary_icd10"                         │
│    ✓ Check: permitted_data.clinical.primary_diagnosis│
│    ✓ Justification: Verify medical necessity         │
│    ✓ Risk level: MEDIUM                              │
│    → ALLOW                                           │
│                                                      │
│ 3. "medications.current_list"                        │
│    ✓ Check: permitted_data.clinical.current_meds     │
│    ✓ Justification: Drug interaction check           │
│    ✓ Risk level: MEDIUM                              │
│    → ALLOW                                           │
│                                                      │
│ 4. "allergies.medication_allergies"                  │
│    ✓ Check: permitted_data.clinical.allergies        │
│    ✓ Justification: Safety screening                 │
│    ✓ Risk level: MEDIUM                              │
│    → ALLOW                                           │
│                                                      │
│ Aggregate risk score: 45/100 → MEDIUM tier           │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 7: Action Authorization Evaluation             │
│                                                      │
│ Requested action: "draft_prior_authorization_request"│
│                                                      │
│ Check against action_catalog.yaml:                   │
│ ✓ Action type: prior_auth.draft_request              │
│ ✓ Risk level: MEDIUM                                 │
│ ✓ Requires approval: NO                              │
│ ✓ User role authorized: YES (attending_physician)    │
│ → ALLOW                                              │
│                                                      │
│ Note: If action was "submit_to_payer":               │
│ ✗ Requires approval: YES                             │
│ ✗ Approver: medical_director (if >$10K)              │
│ → REQUIRE_APPROVAL                                   │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 8: Contextual Validation                       │
│                                                      │
│ Additional checks:                                   │
│ ✓ Patient consent status: Active                    │
│ ✓ VIP patient flag: No                              │
│ ✓ Sealed record flag: No                            │
│ ✓ After-hours access: No (14:23 UTC, business hours)│
│ ✓ Medication-diagnosis alignment: Ozempic indicated  │
│   for Type 2 Diabetes (E11.9) ✓                      │
│                                                      │
│ No contextual violations detected                    │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 9: Confidence Scoring                          │
│                                                      │
│ Evidence Coverage Method:                            │
│                                                      │
│ Claims to verify:                                    │
│ 1. "Patient has diabetes" → ✓ verified (ICD-10)     │
│ 2. "Patient's age is 52" → ✓ verified (DOB calc)    │
│ 3. "Patient takes Metformin" → ✓ verified (med list)│
│ 4. "No known drug allergies" → ✓ verified (empty)   │
│                                                      │
│ Confidence Score: 4/4 = 1.00 (100%)                  │
│ Threshold: 0.85 for auto-approval                   │
│ → EXCEEDS threshold, no human review needed          │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 10: Risk-Tier Enforcement Profile              │
│                                                      │
│ Risk Tier: MEDIUM                                    │
│                                                      │
│ Enforcement Profile Applied:                         │
│ - Latency budget: 100ms                              │
│ - Dual approval: Not required                        │
│ - Logging mode: Detailed                             │
│ - Human gate SLA: 60 minutes (if needed)             │
│                                                      │
│ Time elapsed: 87ms ✓ (under budget)                  │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 11: Decision Finalization                      │
│                                                      │
│ Decision: ALLOW                                      │
│                                                      │
│ Rationale:                                           │
│ - All requested data is minimum necessary            │
│ - All data justified for prior auth workflow         │
│ - No prohibited data requested                       │
│ - Action authorized for user role                    │
│ - No approval triggers activated                     │
│ - Confidence score exceeds threshold                 │
│ - Risk tier: acceptable for auto-approval            │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 12: Decision Object Generation                 │
│                                                      │
│ Generate decision_schema.json object:                │
│ {                                                    │
│   "decision_id": "550e8400-...",                     │
│   "decision": "ALLOW",                               │
│   "timestamp": "2026-01-18T14:23:45.123Z",           │
│   "policy_id": "policy_prior_authorization_v1",      │
│   "identity_chain": {...},                           │
│   "requested_data": [...],                           │
│   "released_data": [...],                            │
│   "blocked_data": [],                                │
│   "requested_actions": [...],                        │
│   "allowed_actions": [...],                          │
│   "risk_assessment": {                               │
│     "risk_tier": "MEDIUM",                           │
│     "confidence_score": 1.00                         │
│   },                                                 │
│   "latency_ms": 87,                                  │
│   "decision_explanation_human_readable": "..."       │
│ }                                                    │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 13: Audit Logging                              │
│                                                      │
│ Write to audit log:                                  │
│ - Decision object (full JSON)                        │
│ - Timestamp, user, patient (anonymized), policy      │
│ - Retention: 6 years (HIPAA requirement)             │
│                                                      │
│ Alert routing:                                       │
│ - Compliance officer: None (routine request)         │
│ - SIEM: Standard log entry                           │
│ - User notification: None needed                     │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 14: Data Release to AI Agent                   │
│                                                      │
│ Guardrail returns to Claude:                         │
│ {                                                    │
│   "status": "ALLOWED",                               │
│   "data": {                                          │
│     "patient_age": 52,                               │
│     "primary_diagnosis": "E11.9",                    │
│     "current_medications": [                         │
│       {"name": "Metformin", "dose": "1000mg BID"}    │
│     ],                                               │
│     "medication_allergies": []                       │
│   },                                                 │
│   "permitted_actions": [                             │
│     "draft_prior_authorization_request"              │
│   ]                                                  │
│ }                                                    │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 15: AI Agent Processing                        │
│                                                      │
│ Claude receives approved data:                       │
│ - Analyzes: 52yo patient with T2DM                   │
│ - Checks: Ozempic indicated for T2DM ✓               │
│ - Verifies: No contraindications with Metformin ✓    │
│ - Generates: Prior auth draft                        │
│                                                      │
│ Draft output:                                        │
│ "Prior authorization request for Ozempic 1mg for     │
│ patient with Type 2 Diabetes Mellitus (ICD-10: E11.9)│
│ currently on Metformin. No known drug allergies.     │
│ Clinically appropriate for diabetes management."     │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│ Step 16: Response to User                           │
│                                                      │
│ Epic EHR displays to Dr. Smith:                      │
│ ┌──────────────────────────────────────────┐        │
│ │ Prior Authorization Draft                 │        │
│ │                                           │        │
│ │ [AI-generated content shown]              │        │
│ │                                           │        │
│ │ [Review] [Edit] [Submit for Approval]    │        │
│ └──────────────────────────────────────────┘        │
│                                                      │
│ Note: Submission still requires human approval       │
│ (different Guardrail policy evaluation)              │
└──────┬──────────────────────────────────────────────┘
       │
       ▼
┌─────────────┐
│     END     │
│  (87ms)     │
└─────────────┘
```

---

## Alternative Flow: BLOCK Scenario

What if Claude had requested "complete_medical_history" instead?

**Modified Step 6:**

```
┌─────────────────────────────────────────────────────┐
│ Step 6: PHI Access Evaluation (BLOCK scenario)      │
│                                                      │
│ Requested: "complete_medical_history"                │
│                                                      │
│ Check against policy:                                │
│ ✗ Field: prohibited_data.complete_medical_history    │
│ ✗ Exception: NONE                                    │
│ ✗ Reason: Exceeds minimum necessary                  │
│ → BLOCK                                              │
│                                                      │
│ Decision: BLOCK (42ms latency)                       │
│                                                      │
│ Enforcement:                                         │
│ - Request blocked immediately                        │
│ - Compliance officer alerted                         │
│ - Violation logged                                   │
│ - User risk score incremented                        │
│                                                      │
│ Response to AI agent:                                │
│ {                                                    │
│   "status": "BLOCKED",                               │
│   "reason": "Complete medical history exceeds        │
│              minimum necessary for prior auth",      │
│   "permitted_alternative": "Request specific data    │
│                            elements (age, diagnosis, │
│                            current meds, allergies)" │
│ }                                                    │
└──────────────────────────────────────────────────────┘
```

---

## Alternative Flow: REQUIRE_APPROVAL Scenario

What if the medication was a $750K specialty drug (off-label use)?

**Modified Steps 7-10:**

```
┌─────────────────────────────────────────────────────┐
│ Step 7: Action Authorization (APPROVAL scenario)    │
│                                                      │
│ Requested: "submit_prior_authorization_to_payer"     │
│                                                      │
│ Check against policy:                                │
│ ✓ Action type: prior_auth.submit_to_payer            │
│ ⚠ Risk level: HIGH                                   │
│ ⚠ Requires approval: YES                             │
│ ⚠ Cost: $750,000/year (exceeds $10K threshold)       │
│ ⚠ Off-label use: YES                                 │
│ → REQUIRE_APPROVAL                                   │
│                                                      │
│ Required approvers:                                  │
│ - medical_director (high cost)                       │
│ - prior_auth_specialist (dual approval)              │
│                                                      │
│ SLA: 2880 minutes (48 hours)                         │
│ Escalation if timeout: BLOCK                         │
└──────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Step 11: Decision (REQUIRE_APPROVAL)                │
│                                                      │
│ Decision: REQUIRE_APPROVAL                           │
│                                                      │
│ Rationale:                                           │
│ - Treatment cost ($750K) far exceeds threshold       │
│ - Off-label use requires medical director review     │
│ - Dual approval necessary for risk mitigation        │
│                                                      │
│ Routing:                                             │
│ - Create approval ticket in ServiceNow               │
│ - Email medical director + prior auth specialist     │
│ - CC: compliance officer (high-value treatment)      │
│ - Set SLA timer: 48 hours                            │
│                                                      │
│ User notification:                                   │
│ "Prior authorization draft created. Submission       │
│ requires medical director approval (high-cost        │
│ treatment). Routed to Dr. Johnson and Prior Auth     │
│ team. Expected response: 48 hours."                  │
└──────────────────────────────────────────────────────┘
```

---

## Data Minimization Example

**Scenario:** AI requests more data than shown to user

**User sees:** "Check coverage for Ozempic"

**AI internally requests:**
- patient_demographics.age ✓
- patient_demographics.date_of_birth ← Excessive (age sufficient)
- diagnoses.primary_icd10 ✓
- diagnoses.complete_problem_list ← Excessive (only primary needed)
- medications.current_list ✓
- complete_medical_history ← Excessive (violates minimum necessary)

**Guardrail processing:**

```
Released to AI:
- patient_demographics.age: 52
- diagnoses.primary_icd10: "E11.9"
- medications.current_list: ["Metformin 1000mg BID"]

Blocked from AI (logged):
- patient_demographics.date_of_birth
  Reason: Age sufficient, DOB is direct identifier
  Alternative: Use age instead

- diagnoses.complete_problem_list
  Reason: Exceeds minimum necessary
  Alternative: Primary diagnosis sufficient

- complete_medical_history
  Reason: Violates HIPAA minimum necessary
  Alternative: Request specific relevant data elements
```

**Result:** AI receives only minimum necessary data, but can still complete the task.

---

## Performance Characteristics

### Latency Breakdown (Typical)

| Step | Time | Cumulative |
|------|------|------------|
| 1-2. User → AI request | ~5ms | 5ms |
| 3. Intercept | <1ms | 6ms |
| 4. Identity validation | ~10ms | 16ms |
| 5. Policy loading (cached) | ~2ms | 18ms |
| 6. PHI evaluation (4 fields) | ~20ms | 38ms |
| 7. Action evaluation | ~15ms | 53ms |
| 8. Context validation | ~10ms | 63ms |
| 9. Confidence scoring | ~12ms | 75ms |
| 10. Risk tier check | ~3ms | 78ms |
| 11-12. Decision + object gen | ~5ms | 83ms |
| 13. Audit logging (async) | ~1ms | 84ms |
| 14. Data release | ~3ms | 87ms |

**Total: 87ms** (well under 100ms MEDIUM tier budget)

### Throughput

- **Single instance:** 500-1000 decisions/second
- **Clustered:** 10,000+ decisions/second
- **Bottlenecks:** Database writes (audit logs), external API calls (identity validation)

### Caching Strategy

**What's cached:**
- Policy files (loaded once per version)
- PHI classification tags
- Action catalog
- User role mappings (5-min TTL)

**What's NOT cached:**
- Patient consent status (always real-time)
- VIP/sealed record flags (always real-time)
- Identity validation (always real-time)
- Decision objects (always generated fresh)

---

## Error Handling

### Fail-Secure Principle

**If Guardrail cannot make a decision → BLOCK**

Examples:
- Policy file corrupted → BLOCK
- Cannot reach identity provider → BLOCK
- Database connection lost → BLOCK (but log locally)
- Unknown PHI field requested → BLOCK
- Policy conflict detected → BLOCK + alert

**Exception:** If Guardrail itself is down, system can be configured to:
1. **Failover to backup Guardrail instance** (preferred)
2. **Block all AI requests** (safest)
3. **Allow with enhanced logging** (emergency mode, requires executive approval)

### Graceful Degradation

If non-critical components fail:
- SIEM connection lost → Continue but log locally
- Approval system (ServiceNow) down → Email fallback
- Analytics/reporting down → Continue enforcement, queue reports

---

## Integration Patterns

### Pattern 1: Proxy Mode

```
AI Agent → Guardrail → Protected Resource

Guardrail acts as proxy/gateway
All requests routed through Guardrail
Simplest integration
```

### Pattern 2: Sidecar Mode

```
AI Agent ←→ Guardrail (sidecar)
   ↓
Protected Resource

Guardrail deployed alongside AI agent
AI calls Guardrail before resource access
More flexible
```

### Pattern 3: Service Mesh

```
AI Agent → Service Mesh (with Guardrail policies) → Resource

Guardrail policies deployed to Istio/Linkerd
Native cloud-native integration
Most scalable
```

---

## Compliance Artifacts Generated

### Real-Time
1. **Decision object** (JSON) - Immediate
2. **Audit log entry** (CSV/JSON) - Immediate
3. **Alert** (if violation) - Immediate

### Batch/Scheduled
1. **Compliance reports** - Weekly
2. **Risk trend analysis** - Monthly
3. **Policy effectiveness metrics** - Monthly

### On-Demand
1. **OCR audit package** - Export all decisions for date range
2. **User activity report** - All decisions by specific user
3. **Violation investigation** - Deep dive on specific incident

---

## Next Steps

- **For Implementation:** Review `/examples` for real decision outputs
- **For Customization:** See `policy_prior_authorization_v1.yaml` comments
- **For Architecture Questions:** Contact architecture@defiantindustries.com

---

**Version:** 1.0  
**Last Updated:** January 18, 2026
