# Guardrail for AI Prior Authorization
**Healthcare Policy Pack v1.1 — Reference Implementation**

**Category:** Policy Enforcement Middleware for Autonomous AI Systems in Regulated Environments

Enforce minimum-necessary PHI access for AI agents and generate compliance evidence.

---

## Audience Guide

**Engineers:** Go to [Quick Start](#quick-start-5-minutes) and [Integration Points](#integration-points)  
**Compliance/Legal:** Go to [Compliance Validation](#compliance-validation) and [Examples](#real-decision-examples)  
**Decision-Makers:** Read [What This Is](#what-this-is) and [Stop Conditions](#stop-conditions-youre-done-when)

---

## What This Is

A reference policy pack that enforces HIPAA minimum necessary standard for AI agents processing prior authorizations.

**In 5 minutes, you'll see:**
- AI agent requests complete medical history → **BLOCKED**
- AI agent requests only necessary data → **ALLOWED**
- AI agent tries to submit without approval → **REQUIRES APPROVAL**

**The output:** Audit-ready decision logs proving compliance.

---

## Quick Start (5 Minutes)

### Prerequisites
- Docker OR Python 3.9+
- Sample prior auth request (we provide one)

### Option 1: Run the Demo (Fastest)

```bash
# Clone or download this policy pack
cd policies/healthcare/v1

# Run validation
python validate_policy.py policy_prior_authorization_v1.yaml

# Test with sample requests
python test_policy.py policy_prior_authorization_v1.yaml examples/prior_auth_request.json

# See decision output
cat examples/prior_auth_decision_allow.json
```

### Option 2: Integrate With Your AI Agent

```python
import guardrail

# Load policy
policy = guardrail.load_policy("policy_prior_authorization_v1.yaml")

# AI agent makes request
request = {
    "user_id": "NPI_1234567890",
    "user_role": "attending_physician",
    "agent_id": "claude-healthcare-instance-1",
    "workflow_type": "prior_authorization",
    "requested_data": [
        {"field": "patient_demographics.age"},
        {"field": "diagnoses.primary_icd10"}
    ],
    "requested_actions": [
        {"action": "draft_prior_authorization_request"}
    ]
}

# Guardrail evaluates
decision = guardrail.evaluate(policy, request)

# decision.decision → "ALLOW" | "REQUIRE_APPROVAL" | "BLOCK"
# decision.released_data → what AI can actually access
# decision.blocked_data → what was denied and why
# decision.explanation → human-readable compliance report
```

---

## What's In This Pack

### Core Policy
**`policy_prior_authorization_v1.yaml`** - Reference policy with:
- ✅ HIPAA minimum necessary enforcement
- ✅ 42 CFR Part 2 scope correct (SUD programs only)
- ✅ Action control (not just data governance)
- ✅ Risk-tier enforcement (latency budgets, dual approval)
- ✅ Confidence scoring (evidence coverage method)
- ✅ Identity chain tracking (USER→ROLE→AGENT→POLICY)
- ✅ Decision explainability (designed to support OCR and internal audit workflows)

### Schemas
**`schemas/decision_schema.json`** - Canonical decision object format
- Every decision follows this schema
- Designed to be consumable by audit and compliance teams through a standardized format
- Exportable to CSV/JSON for compliance reviews

### Examples
**`examples/`** - Real decision outputs:
- `prior_auth_decision_allow.json` - Routine request (example latency: 87ms)
- `prior_auth_decision_block.json` - Excessive PHI request (BLOCKED)
- `prior_auth_decision_require_approval.json` - High-cost treatment (REQUIRES APPROVAL)

### Mappings
**`mappings/phi_tags.yaml`** - PHI classification taxonomy  
**`mappings/action_catalog.yaml`** - Healthcare action risk levels

---

## How It Works

### 1. Identity Chain
Every request is traced:
```
USER:NPI_1234567890 → ROLE:attending_physician → AGENT:claude-instance-42 → POLICY:policy_prior_authorization_v1
```

### 2. PHI Access Control
Policy checks:
- **Is this data minimum necessary?** (HIPAA 164.502(b))
- **Does diagnosis justify accessing this medication?**
- **Is protected category PHI requested?** (mental health, SUD, HIV, genetic)

### 3. Action Control
Policy checks:
- **Can AI draft requests?** Yes (low risk)
- **Can AI submit to payer?** No, requires human approval (high risk)
- **Can AI modify clinical records?** No, wrong workflow

### 4. Risk-Based Enforcement
- **LOW risk** (default latency budget: 50ms, configurable): Routine requests
- **MEDIUM risk** (100ms): Standard prior auths
- **HIGH risk** (150ms): Protected category access
- **CRITICAL risk** (200ms): High-cost + dual approval required

### 5. Confidence Scoring
Evidence coverage method:
```
Score = verified_claims / total_claims

Example:
- "Patient has diabetes" → ✓ verified (ICD-10 in EHR)
- "Patient takes metformin" → ✓ verified (active med list)
- "Patient allergic to penicillin" → ✗ not documented
Score: 2/3 = 0.67 → triggers human review (<0.85 threshold)
```

### 6. Decision Explainability
Every decision includes human-readable explanation:
```
BLOCKED: AI agent requested complete medical history for prior authorization.
Per HIPAA minimum necessary standard, this exceeds what's needed.
Compliance officer has been notified of violation attempt.
```

---

## Configuration Knobs

### Approval Thresholds
Edit `human_oversight.mandatory_review`:
```yaml
- trigger: "cost_exceeds_$10000"  # Change threshold here
  reviewer: "medical_director"
```

### Confidence Threshold
Edit `confidence_assessment.thresholds`:
```yaml
high_confidence: 0.85  # Adjust based on your risk tolerance
```

### Latency Budgets
Edit `risk_tier.tier_definitions`:
```yaml
MEDIUM:
  enforcement_profile:
    latency_budget_ms: 100  # Increase if needed
```

### Data Residency
Edit `data_governance.allowed_data_locations`:
```yaml
examples_by_provider:
  AWS: ["us-east-1", "us-west-2"]
  Azure: ["EastUS", "WestUS"]
  On_Premises: ["your_data_center"]
```

---

## Integration Points

### Upstream (AI Agents)
- Claude API (Anthropic MCP)
- ChatGPT API (OpenAI)
- LangChain
- Custom healthcare AI

### Downstream (PHI Sources)
- Epic (HL7 FHIR)
- Cerner
- Athenahealth
- CMS Coverage Database
- Payer APIs

### Lateral (Governance)
- Identity: Okta, Azure AD
- SIEM: Splunk, QRadar
- Ticketing: ServiceNow (for approvals)

---

## Compliance Validation

### For Auditors
This policy pack provides:
1. **Documented enforcement mechanism** (policy_prior_authorization_v1.yaml)
2. **Audit trail format** (decision_schema.json)
3. **Sample decision logs** (examples/)
4. **Risk assessment methodology** (evidence coverage scoring)
5. **Human oversight documentation** (approval workflows)

### OCR Audit Questions
**Q: How do you enforce minimum necessary?**  
A: Policy engine validates every request against minimum necessary rules. See `phi_access_control` section and decision logs.

**Q: Can you prove AI only accessed authorized PHI?**  
A: Every decision log shows `requested_data`, `released_data`, and `blocked_data` with justifications.

**Q: What happens if AI tries unauthorized access?**  
A: Request blocked automatically. Compliance officer alerted. See `prior_auth_decision_block.json` example.

---

## Common Use Cases

### Use Case 1: Routine Prior Auth (ALLOW)
- Physician: "Check coverage for Ozempic for diabetic patient"
- AI requests: age, primary diagnosis, current meds, allergies
- **Decision: ALLOW** - all data is minimum necessary
- Latency: example 87ms

### Use Case 2: Excessive Data Request (BLOCK)
- AI requests: complete medical history + mental health diagnoses
- **Decision: BLOCK** - exceeds minimum necessary
- Compliance officer notified
- Latency: example 42ms

### Use Case 3: High-Cost Treatment (REQUIRE_APPROVAL)
- Physician: "Prior auth for $750K specialty drug (off-label)"
- AI drafts request successfully
- Submission **BLOCKED** until medical director approves
- Routed to medical director + prior auth specialist
- SLA: 48 hours
- Latency: example 156ms

---

## Stop Conditions (You're Done When...)

✅ Policy loads and validates against schema  
✅ Sample request produces valid decision.json  
✅ Decision exports to CSV/JSON for audit review  
✅ All 7 technical fixes applied  
✅ 3 infrastructure additions working (identity chain, risk tiers, explainability)

---

## Scope of This Package

### This starter pack intentionally includes:
- Reference policy for prior authorization
- Canonical schemas for decision, identity, and risk enforcement
- Example decision artifacts
- PHI and action taxonomies
- HIPAA compliance mapping

### This package intentionally excludes:
- Full multi-workflow policy catalogs (clinical documentation, care coordination, billing)
- Enterprise IAM connectors and adapters
- Production deployment tooling (HA, secrets management, tenant isolation)
- Industry-specific regulatory extensions beyond HIPAA

**These are provided through pilot engagements and enterprise deployments.**

---

## Next Steps

### For Testing
1. Run `validate_policy.py` on your customized policy
2. Test with your own prior auth scenarios
3. Review decision logs with compliance team

### For Production
1. Integrate with your EHR (Epic/Cerner FHIR endpoint)
2. Connect to identity provider (Okta/AD)
3. Configure approval routing (ServiceNow/email)
4. Set up SIEM integration (Splunk/QRadar)
5. Train users on approval workflows

### For Compliance
1. Share policy with compliance officer
2. Review decision examples with auditors
3. Prepare for OCR audit using decision logs
4. Document in HIPAA Security Risk Assessment

---

## Troubleshooting

**Policy validation fails**
- Check YAML syntax: `yamllint policy_prior_authorization_v1.yaml`
- Verify all required fields present
- Check field references in `phi_tags.yaml`

**Decision says "BLOCK" when should "ALLOW"**
- Review `prohibited_data` section
- Check if data is in `permitted_data`
- Verify workflow_type matches request

**Latency exceeds budget**
- Review risk_tier thresholds
- Consider increasing latency_budget_ms
- Check if multiple approval triggers firing

**Confidence score always low**
- Verify evidence_refs in request
- Check if EHR fields are being verified
- May need to adjust threshold if data quality varies

---

## Support

**Technical Questions:** support@defiantindustries.com  
**Compliance Questions:** compliance@defiantindustries.com  
**Sales/Pilot:** healthcare-pilots@defiantindustries.com  

**Schedule Demo:** defiantindustries.com/healthcare-demo

---

## License

This policy pack is provided as reference implementation.  
Customize for your organization before production use.

**Regulatory Disclaimer:**  
This repository does not constitute legal or regulatory advice. Organizations are responsible for validating compliance with applicable laws and regulations, and for engaging their compliance, privacy, and legal teams prior to production deployment.

© 2026 Defiant Industries Inc. All rights reserved.

---

**You're 5 minutes away from AI governance that supports audit requirements.**

**Start:** `python validate_policy.py policy_prior_authorization_v1.yaml`
