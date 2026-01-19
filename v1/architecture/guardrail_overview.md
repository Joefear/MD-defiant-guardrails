# Guardrail Architecture Overview

**Independent Policy Enforcement Layer for AI Agents in Regulated Environments**

---

## What Guardrail Is

Guardrail is **middleware** that sits between AI agents and protected data/systems, enforcing organization-specific policies in real-time before AI agents can access data or execute actions.

**Key principle:** Policy enforcement must be independent of the AI provider.

---

## Where Guardrail Sits

```
┌─────────────────────────────────────────────────────────────┐
│                    HUMAN USERS                              │
│        (Physicians, Nurses, Staff, Patients)                │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Natural language requests
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  AI AGENTS                                  │
│  (Claude, ChatGPT, LangChain, Custom)                       │
│                                                             │
│  "Check coverage for Drug X for diabetic patient"           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Agent requests data/actions
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              🛡️ GUARDRAIL MIDDLEWARE 🛡️                      │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  1. Request Interception                           │   │
│  │     "Agent wants complete medical history"         │   │
│  └────────────────────────────────────────────────────┘   │
│                        │                                    │
│                        ▼                                    │
│  ┌────────────────────────────────────────────────────┐   │
│  │  2. Policy Evaluation                              │   │
│  │     Load: policy_prior_authorization_v1.yaml       │   │
│  │     Check: Does request violate minimum necessary? │   │
│  │     Check: Is action permitted for this workflow?  │   │
│  │     Check: Does risk tier require human approval?  │   │
│  └────────────────────────────────────────────────────┘   │
│                        │                                    │
│                        ▼                                    │
│  ┌────────────────────────────────────────────────────┐   │
│  │  3. Decision + Enforcement                         │   │
│  │     ALLOW / REQUIRE_APPROVAL / BLOCK               │   │
│  └────────────────────────────────────────────────────┘   │
│                        │                                    │
│                        ▼                                    │
│  ┌────────────────────────────────────────────────────┐   │
│  │  4. Audit Logging                                  │   │
│  │     Generate decision_schema.json object           │   │
│  │     Log: what was requested, allowed, blocked      │   │
│  │     Export: compliance-ready reports               │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        │ Scoped data/approved actions only
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              PROTECTED RESOURCES                            │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  EHR Systems │  │  Claims DB   │  │  Lab Systems │    │
│  │  (Epic/Cerner)│  │              │  │              │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │  Patient     │  │  Pharmacy    │  │  Payer APIs  │    │
│  │  Portal      │  │  Systems     │  │              │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

**Critical insight:** Guardrail is the **only path** between AI agents and protected resources. All requests flow through policy enforcement.

---

## What Guardrail Intercepts

### 1. Data Access Requests
**Example:** AI agent requests patient data

```python
# AI agent request (intercepted by Guardrail)
{
  "workflow_type": "prior_authorization",
  "requested_data": [
    "patient_demographics.age",
    "diagnoses.primary_icd10",
    "complete_medical_history"  # ← Excessive
  ]
}
```

**Guardrail evaluates:**
- Is this data minimum necessary for prior auth?
- Does user role permit this access?
- Is patient consent valid?
- Does data include protected categories (mental health, HIV, genetic)?

### 2. Action Execution Requests
**Example:** AI agent wants to submit prior authorization

```python
# AI agent action request (intercepted by Guardrail)
{
  "workflow_type": "prior_authorization",
  "requested_actions": [
    "draft_prior_authorization_request",      # Low risk
    "submit_prior_authorization_to_payer"     # High risk
  ]
}
```

**Guardrail evaluates:**
- Is agent permitted to execute this action?
- Does this action require human approval?
- What's the risk tier? (cost, off-label use, experimental)
- Has approval been obtained if required?

### 3. Identity Chain
**Example:** Who's behind this request?

```python
# Identity chain (verified by Guardrail)
{
  "human_user_id": "NPI_1234567890",
  "user_role": "attending_physician",
  "ai_agent_id": "claude-sonnet-4-instance-42",
  "system_of_origin": "epic_smartfhir"
}
```

**Guardrail validates:**
- Is user authenticated?
- Does role match permissions?
- Is AI agent registered?
- Complete audit trail: USER → ROLE → AGENT → POLICY

---

## What Guardrail Emits

### Primary Output: Decision Object

Every Guardrail decision follows `decision_schema.json`:

```json
{
  "decision_id": "550e8400-e29b-41d4-a716-446655440001",
  "decision": "ALLOW | REQUIRE_APPROVAL | BLOCK",
  "timestamp": "2026-01-18T14:23:45.123Z",
  
  "identity_chain": {
    "human_user_id": "NPI_1234567890",
    "user_role": "attending_physician",
    "ai_agent_id": "claude-instance-42"
  },
  
  "requested_data": [...],
  "released_data": [...],    // What AI actually got
  "blocked_data": [...],     // What was denied + why
  
  "requested_actions": [...],
  "allowed_actions": [...],
  "blocked_actions": [...],
  
  "risk_assessment": {
    "risk_tier": "MEDIUM",
    "confidence_score": 0.95
  },
  
  "human_gate": {
    "required": false | true,
    "reason": "...",
    "reviewer_role": "medical_director"
  },
  
  "decision_explanation_human_readable": "..."
}
```

### Secondary Outputs

1. **Audit Logs** (CSV/JSON export)
   - All decisions for compliance review
   - Filterable by user, workflow, date, outcome
   - Retention: 6 years (HIPAA requirement)

2. **Compliance Reports**
   - Weekly/monthly summaries
   - Violation attempt tracking
   - Human approval metrics
   - Risk tier distribution

3. **Alerts**
   - Real-time to compliance officers (violations)
   - Escalation notifications (approvals needed)
   - Anomaly detection (unusual access patterns)

---

## Why Guardrail Is Independent of the AI Model

### The Trust Problem

**Scenario:** Hospital asks OpenAI or Anthropic:
> "How do we know your AI only accesses minimum necessary PHI?"

**AI Provider Answer:**
> "Our model is trained to respect privacy. We have a HIPAA BAA. Trust us."

**Hospital's Problem:**
- Auditors don't accept "trust us"
- No independent verification
- AI provider can't be judge and jury of own compliance

### The Guardrail Solution

**Independence = Verifiability**

1. **Separate Codebase**
   - Guardrail code ≠ AI model code
   - Open source = auditable
   - Customer can fork and customize

2. **Separate Policy Definition**
   - Policies written by customer, not AI provider
   - Customer controls enforcement rules
   - Can be stricter than regulatory minimums

3. **Separate Decision Authority**
   - Guardrail makes ALLOW/BLOCK decisions
   - AI model cannot override
   - Enforcement happens before AI gets data

4. **Separate Audit Trail**
   - Guardrail logs ≠ AI provider logs
   - Customer owns compliance evidence
   - Can export to third-party auditors

**Analogy:**
- AI model = Employee asking for file access
- Guardrail = Access control system enforcing permissions
- You don't ask employee to enforce their own permissions

---

## Technical Architecture

### Integration Model

**Guardrail as Middleware:**

```
┌──────────────────────────────────────────────────────┐
│  Application Layer                                   │
│  ├─ EHR UI (Epic, Cerner)                            │
│  ├─ Patient Portal                                   │
│  └─ Administrative Systems                           │
└──────────────────┬───────────────────────────────────┘
                   │
                   │ API calls to AI
                   ▼
┌──────────────────────────────────────────────────────┐
│  AI Agent Layer                                      │
│  ├─ Claude (Anthropic MCP)                           │
│  ├─ ChatGPT (OpenAI API)                             │
│  ├─ LangChain Agents                                 │
│  └─ Custom AI Applications                           │
└──────────────────┬───────────────────────────────────┘
                   │
                   │ Data/action requests
                   ▼
┌──────────────────────────────────────────────────────┐
│  🛡️ GUARDRAIL MIDDLEWARE 🛡️                          │
│                                                       │
│  Policy Engine:                                       │
│  ├─ Load policies (policy_*.yaml)                    │
│  ├─ Evaluate requests against rules                  │
│  ├─ Apply risk-tier enforcement                      │
│  └─ Generate decision object                         │
│                                                       │
│  Integration:                                         │
│  ├─ Identity Provider (Okta, Azure AD)               │
│  ├─ SIEM (Splunk, QRadar)                            │
│  └─ Approval System (ServiceNow)                     │
└──────────────────┬───────────────────────────────────┘
                   │
                   │ Approved requests only
                   ▼
┌──────────────────────────────────────────────────────┐
│  Data Layer                                          │
│  ├─ EHR Database (Epic, Cerner)                      │
│  ├─ Claims Systems                                   │
│  ├─ Lab/Imaging (PACS, LIS)                          │
│  └─ External APIs (Payers, Pharmacies)              │
└──────────────────────────────────────────────────────┘
```

### Deployment Models

**1. On-Premises**
- Guardrail deployed in customer data center
- Full control over policy enforcement
- No data leaves customer network
- Ideal for: Large health systems, high-security orgs

**2. Private Cloud (Customer VPC)**
- Guardrail in customer's AWS/Azure/GCP tenant
- Customer-managed keys
- Integration with customer IAM
- Ideal for: Cloud-native healthcare orgs

**3. Hybrid**
- Guardrail on-prem, AI in cloud
- Policy enforcement at network boundary
- Data minimization before cloud transmission
- Ideal for: Gradual cloud migration

**4. SaaS (Managed Guardrail)**
- Defiant-hosted, HIPAA-compliant infrastructure
- Multi-tenant with data isolation
- Fastest time to value
- Ideal for: Startups, small practices

---

## Key Design Principles

### 1. Policy-as-Code
- Policies defined in human-readable YAML
- Version controlled (Git)
- Testable before deployment
- Auditable policy changes

### 2. Zero Trust for AI
- Every request evaluated independently
- No "trusted" AI agents
- Continuous verification, never assume

### 3. Defense in Depth
- Multiple policy layers (data + actions + risk)
- Redundant checks (identity + context + purpose)
- Fail-secure (block if uncertain)

### 4. Auditability First
- Every decision logged
- Complete evidence chain
- Compliance-ready by design

### 5. Human Oversight When Needed
- Configurable approval gates
- SLA-driven escalation
- Clear approval chains

---

## Comparison to Alternatives

| Approach | Independence | Customizable | Audit Trail | Healthcare-Ready |
|----------|-------------|--------------|-------------|------------------|
| **Trust AI Provider** | ❌ No | ❌ No | ❌ Insufficient | ⚠️ Claims only |
| **Build In-House** | ✅ Yes | ✅ Yes | ⚠️ If done right | ⚠️ 6-12 months |
| **IAM/Access Control** | ⚠️ Partial | ⚠️ Limited | ⚠️ System logs only | ❌ Not AI-aware |
| **API Gateway** | ⚠️ Partial | ✅ Yes | ⚠️ HTTP logs only | ❌ No policy engine |
| **GRC Platform** | ❌ Post-hoc | ❌ Rigid | ✅ Yes | ❌ Manual |
| **Guardrail** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Purpose-built |

---

## Scaling Considerations

### Performance
- **Latency:** <100ms policy evaluation (typical)
- **Throughput:** Thousands of decisions/second
- **Caching:** Policy rules cached, data access real-time

### Reliability
- **High Availability:** Multi-AZ deployment, 99.9% uptime
- **Failover:** Backup policy enforcement nodes
- **Graceful Degradation:** Fail-secure (block if Guardrail down)

### Extensibility
- **New Policies:** Add `policy_*.yaml` files
- **New Workflows:** Extend action catalog
- **New Regulations:** Update PHI classification tags
- **New AI Providers:** Standard integration interface

---

## Who Uses Guardrail

### Primary Users
1. **Healthcare Organizations** deploying AI for clinical/administrative workflows
2. **Health AI Startups** needing enterprise-ready governance for hospital sales
3. **Payers/Insurers** using AI for claims, prior auth, member services
4. **Clinical Trial Organizations** using AI for recruitment, monitoring, regulatory

### Expansion Markets
- **Financial Services** (SOX, PCI-DSS compliance for AI)
- **Government** (FedRAMP, CMMC for AI in defense/civilian agencies)
- **Legal** (Attorney-client privilege enforcement for legal AI)
- **Any Regulated Industry** deploying AI with compliance requirements

---

## Strategic Positioning

**Guardrail is not:**
- ❌ An AI model
- ❌ A compliance checklist
- ❌ A single-industry solution

**Guardrail is:**
- ✅ Governance infrastructure for autonomous AI systems
- ✅ Policy enforcement middleware
- ✅ Multi-industry platform (healthcare is proof domain)

**Analogy:**
- Firewalls didn't replace networks—they made networks safe
- Guardrail doesn't replace AI—it makes AI deployable in regulated environments

---

## Next Steps

### For Technical Evaluation
1. Review `policy_prior_authorization_v1.yaml` - See production policy
2. Examine `decision_schema.json` - Understand decision format
3. Read `data_flow.md` - Deep dive on request/response flow

### For Business Evaluation
1. Review example decisions in `/examples` - See real outputs
2. Read PACKAGE_INDEX.md - Understand complete offering
3. Contact: healthcare-pilots@defiantindustries.com

### For Compliance/Legal
1. Review HIPAA policy mappings in `/mappings/phi_tags.yaml`
2. Examine audit trail format in `decision_schema.json`
3. Review decision explainability templates

---

**Version:** 1.0  
**Last Updated:** January 18, 2026  
**Contact:** info@defiantindustries.com  

**Learn more:** [GitHub Repository] | [Schedule Demo]
