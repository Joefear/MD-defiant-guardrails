# HIPAA Compliance Mapping

**How Guardrail Features Map to HIPAA Requirements**

This document shows how specific Guardrail capabilities satisfy HIPAA Security and Privacy Rule requirements.

---

## Quick Reference Table

| Guardrail Feature | HIPAA Clause | How Enforced | Evidence Location |
|-------------------|--------------|--------------|-------------------|
| **Minimum Necessary** | 45 CFR 164.502(b) | PHI scoping rules in policy files | `policy_*.yaml` → `permitted_data` section |
| **Audit Controls** | 45 CFR 164.312(b) | Decision schema + logging | `decision_schema.json` → audit logs |
| **Access Control** | 45 CFR 164.312(a) | Identity chain validation | `identity_chain_schema.yaml` |
| **Integrity Controls** | 45 CFR 164.312(c) | Fact-checking + explainability | `decision_explainability_schema.yaml` |
| **Transmission Security** | 45 CFR 164.312(e) | Encryption enforcement | Policy → `data_governance.encryption` |
| **Unique User ID** | 45 CFR 164.312(a)(2)(i) | Identity chain tracking | Decision object → `identity_chain.human_user_id` |
| **Emergency Access** | 45 CFR 164.312(a)(2)(ii) | Risk tier escalation | `risk_tier_schema.yaml` |
| **Automatic Logoff** | 45 CFR 164.312(a)(2)(iii) | Session termination | Policy → `session_timeout` |
| **Person/Entity Auth** | 45 CFR 164.312(d) | Identity validation | `identity_chain.user_role` verification |

---

## Detailed Mappings

### 1. Minimum Necessary Standard (§164.502(b))

**HIPAA Requirement:**
> "A covered entity must make reasonable efforts to limit protected health information to the minimum necessary to accomplish the intended purpose of the use, disclosure, or request."

**Guardrail Implementation:**

**Policy File Definition:**
```yaml
# policy_prior_authorization_v1.yaml

permitted_data:
  clinical_information:
    - field: "primary_diagnosis"
      justification: "Determine medical necessity"
    
    - field: "current_medications"
      justification: "Drug interaction check"

prohibited_data:
  - field: "complete_medical_history"
    reason: "Exceeds minimum necessary"
```

**Runtime Enforcement:**
- AI requests complete medical history → **BLOCKED**
- AI requests only primary diagnosis → **ALLOWED**
- Every decision logs: what was requested, what was allowed, what was blocked, why

**Evidence for Auditors:**
- Policy file shows minimum necessary rules
- Decision logs show enforcement in action
- Blocked access attempts prove active controls

---

### 2. Audit Controls (§164.312(b))

**HIPAA Requirement:**
> "Implement hardware, software, and/or procedural mechanisms that record and examine activity in information systems that contain or use electronic protected health information."

**Guardrail Implementation:**

**Decision Object Schema:**
```json
{
  "decision_id": "550e8400-...",
  "timestamp": "2026-01-18T14:23:45.123Z",
  "identity_chain": {
    "human_user_id": "NPI_1234567890",
    "user_role": "attending_physician",
    "ai_agent_id": "claude-instance-42"
  },
  "requested_data": [...],
  "released_data": [...],
  "blocked_data": [...],
  "decision": "ALLOW | REQUIRE_APPROVAL | BLOCK"
}
```

**What Gets Logged:**
- **Who:** Complete identity chain (user → role → agent → policy)
- **What:** Exact PHI fields requested and released
- **When:** ISO 8601 timestamp with timezone
- **Why:** Policy rules that authorized/blocked
- **How:** Decision explanation in human-readable format

**Evidence for Auditors:**
- Every AI interaction with PHI logged
- Tamper-evident (digital signatures)
- Retention: Configurable to meet HIPAA/state requirements (commonly 6+ years)
- Exportable: CSV, JSON, PDF for compliance reviews

---

### 3. Access Control (§164.312(a))

**HIPAA Requirement:**
> "Implement technical policies and procedures for electronic information systems that maintain electronic protected health information to allow access only to those persons or software programs that have been granted access rights."

**Guardrail Implementation:**

**Identity Chain Validation:**
```yaml
# identity_chain_schema.yaml

required_fields:
  - human_user_id      # Who initiated the request
  - user_role          # What permissions they have
  - ai_agent_id        # Which AI is requesting
  - system_of_origin   # Where request came from

enforcement:
  block_if_missing: true
  verify_against: "identity_provider (Okta, Azure AD)"
```

**Role-Based Access Control:**
- Policies specify which roles can access which PHI
- AI agent cannot override user permissions
- Every access requires valid identity chain

**Evidence for Auditors:**
- Identity chain in every decision log
- Role verification before data release
- Audit trail: USER → ROLE → AGENT → POLICY → DATA

---

### 4. Integrity Controls (§164.312(c))

**HIPAA Requirement:**
> "Implement policies and procedures to protect electronic protected health information from improper alteration or destruction."

**Guardrail Implementation:**

**For AI-Generated Content:**
- Fact-checking against source EHR data
- Hallucination detection (unverified statements flagged)
- Human review required before finalization

**For Data Access:**
- Read-only by default
- Write actions require explicit approval
- Version control for all modifications

**Evidence for Auditors:**
- Decision explainability shows fact verification
- Hallucination checks logged
- Human approval chain documented

**Example from `policy_clinical_documentation_v1.yaml`:**
```yaml
hallucination_safeguards:
  - control: "fact_check_against_source_data"
    rule: "Every clinical finding must exist in source EHR"
    enforcement: "Block if unverified facts detected"
```

---

### 5. Transmission Security (§164.312(e))

**HIPAA Requirement:**
> "Implement technical security measures to guard against unauthorized access to electronic protected health information that is being transmitted over an electronic communications network."

**Guardrail Implementation:**

**Policy Enforcement:**
```yaml
data_governance:
  encryption:
    in_transit: "TLS_1.3_minimum"
    at_rest: "AES_256"
    key_management: "customer_managed_keys"
  
  transmission_controls:
    - block_unencrypted_endpoints
    - verify_certificate_validity
    - log_all_transmissions
```

**Runtime Checks:**
- AI cannot send PHI to unencrypted endpoints
- Certificate validation before transmission
- Data residency rules enforced

---

### 6. Unique User Identification (§164.312(a)(2)(i))

**HIPAA Requirement:**
> "Assign a unique name and/or number for identifying and tracking user identity."

**Guardrail Implementation:**

**Every Decision Logs:**
```json
{
  "identity_chain": {
    "human_user_id": "NPI_1234567890",  ← Unique identifier
    "user_role": "attending_physician",
    "ai_agent_id": "claude-instance-42",  ← Unique AI agent ID
    "session_id": "sess_abc123"          ← Unique session
  }
}
```

**Uniqueness Guarantee:**
- Human users: NPI or employee ID
- AI agents: Instance-specific identifier
- Sessions: Unique per interaction

---

### 7. Emergency Access Procedure (§164.312(a)(2)(ii))

**HIPAA Requirement:**
> "Establish (and implement as needed) procedures for obtaining necessary electronic protected health information during an emergency."

**Guardrail Implementation:**

**Risk Tier Escalation:**
```yaml
risk_tier:
  CRITICAL:
    enforcement_profile:
      emergency_override: "possible_with_executive_approval"
      enhanced_logging: "forensic_mode"
      immediate_notification: "compliance_officer + legal"
```

**Emergency Mode:**
- Can be activated by authorized personnel
- Requires documented emergency justification
- Enhanced audit trail during emergency access
- Post-emergency review mandatory

---

### 8. Automatic Logoff (§164.312(a)(2)(iii))

**HIPAA Requirement:**
> "Implement electronic procedures that terminate an electronic session after a predetermined time of inactivity."

**Guardrail Implementation:**

**Session Management:**
```yaml
session_control:
  timeout_minutes: 15        # Configurable per organization
  require_reauthentication: true
  terminate_on_policy_change: true
```

**Per-Request Authorization:**
- Each AI request requires fresh authentication
- Sessions automatically terminated after timeout
- Agent must re-authenticate for each patient context switch

---

### 9. Person or Entity Authentication (§164.312(d))

**HIPAA Requirement:**
> "Implement procedures to verify that a person or entity seeking access to electronic protected health information is the one claimed."

**Guardrail Implementation:**

**Multi-Layer Authentication:**
1. **Human User:** Verified via identity provider (Okta, Azure AD, etc.)
2. **AI Agent:** Verified via agent registry
3. **Request Context:** Verified against session token

**Integration:**
```yaml
identity_validation:
  sources:
    - "okta_saml"
    - "azure_ad_oauth2"
    - "on_prem_ldap"
  
  verification_steps:
    - authenticate_user
    - verify_role_membership
    - validate_agent_registration
    - check_session_validity
```

---

## Administrative Safeguards

### Access Management (§164.308(a)(4))

**Guardrail Support:**

| HIPAA Requirement | Guardrail Implementation |
|-------------------|-------------------------|
| Authorize access based on role | Policy files define role-based permissions |
| Implement procedures to determine access | Identity chain validation + policy evaluation |
| Grant minimum necessary | PHI scoping in permitted_data sections |
| Review and modify access | Policy version control + change logs |

---

## Compliance Reporting

### What Guardrail Provides for OCR Audits

**1. Documented Policies**
- Location: `policy_*.yaml` files
- Version controlled (Git)
- Signed and dated approvals section

**2. Technical Controls Evidence**
- Location: `schemas/` folder
- Shows enforcement mechanisms
- Architecture diagrams in `/architecture`

**3. Audit Trails**
- Location: Decision logs (exportable)
- Format: CSV, JSON, PDF
- Content: Complete evidence chain

**4. Effectiveness Metrics**
- Policy violation attempts (and blocks)
- Human approval rates
- Risk tier distribution
- Compliance over time

---

## Common Audit Questions → Guardrail Answers

**Q: How do you enforce minimum necessary?**
**A:** Policy files define permitted PHI per workflow. Runtime engine blocks requests exceeding those rules. See decision logs for evidence.

**Q: How do you prove only authorized personnel accessed PHI?**
**A:** Identity chain in every decision log shows: user ID, role, verification method, timestamp.

**Q: What happens if someone tries unauthorized access?**
**A:** Request blocked immediately. Violation logged. Compliance officer alerted. See blocked decision examples.

**Q: How do you track PHI disclosures?**
**A:** Every data release logged in decision object with: what data, to whom, for what purpose, under which policy.

**Q: Can you demonstrate your audit controls work?**
**A:** Yes. Export decision logs for date range. Filter by user/patient/workflow. Show blocked attempts, approvals, access patterns.

---

## State-Specific Requirements

Guardrail policies can be extended for state-specific laws:

**California (CMIA):**
- Add explicit consent checks
- Enhanced patient notification
- Stricter disclosure rules

**New York (PHL Article 27-F):**
- HIV status special handling
- Additional consent requirements
- Enhanced privacy protections

**Configuration:**
```yaml
state_compliance:
  california_cmia:
    enabled: true
    consent_type: "explicit_per_disclosure"
  
  new_york_phl_27f:
    enabled: true
    hiv_enhanced_protections: true
```

---

## Gap Analysis

### What Guardrail Does NOT Replace

Guardrail is **policy enforcement infrastructure**, not:

❌ Complete HIPAA compliance program (you still need policies, training, BAAs)  
❌ Physical safeguards (facility security, workstation controls)  
❌ Breach notification procedures (you need incident response plans)  
❌ Business associate management (you still need contracts)  

Guardrail handles: **Technical safeguards for AI systems accessing PHI**

---

## Certification Support

**HITRUST:**
- Guardrail supports HITRUST Common Security Framework (CSF) controls
- Decision logs map to evidence requirements
- Policies align with HITRUST assessment criteria

**SOC 2:**
- Audit trails support SOC 2 Type II requirements
- Access controls demonstrate security principle adherence
- Continuous monitoring for availability + security

---

## Next Steps

### For Compliance Officers

1. **Review** example decisions to see audit trail quality
2. **Test** policy enforcement with sample scenarios
3. **Export** decision logs and share with auditors
4. **Customize** policies for organization-specific requirements

### For HIPAA Security Officers

1. **Map** your current risk analysis to Guardrail features
2. **Document** Guardrail as technical safeguard in Security Rule compliance
3. **Integrate** with existing access controls and SIEM
4. **Test** incident response with blocked access scenarios

### For Preparing for OCR Audit

1. **Generate** compliance reports from decision logs
2. **Prepare** policy documentation package
3. **Document** enforcement effectiveness (violations blocked)
4. **Train** staff on Guardrail as part of security awareness

---

## Contact

**Compliance Questions:** compliance@defiantindustries.com  
**Technical Implementation:** support@defiantindustries.com  
**Audit Preparation:** healthcare-pilots@defiantindustries.com

---

**Version:** 1.0  
**Last Updated:** January 18, 2026  
**Reviewed By:** [Legal Counsel] | [Chief Compliance Officer]

**Note:** This mapping is provided for informational purposes. Healthcare organizations should consult with qualified legal and compliance professionals when implementing AI governance systems.
