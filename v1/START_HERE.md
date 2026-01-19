# Start Here: 10-Minute Golden Path

**Welcome to Guardrail Healthcare Starter Pack v1.1**

This guide takes you from "what is this?" to "I understand how it works" in 10 minutes.

---

## The Path

### 1. **Understand What Guardrail Is** (3 minutes)

📖 Open: `architecture/guardrail_overview.md`

**You'll learn:**
- Where Guardrail sits (middleware between AI and protected data)
- What it intercepts (data requests, actions, identity)
- What it emits (auditable decision objects)
- Why independence matters (can't have AI provider police itself)

**Key takeaway:** Guardrail is a control plane for AI in regulated environments.

---

### 2. **See It Work** (2 minutes)

📊 Open: `examples/prior_auth_decision_block.json`

**You'll see:**
- Real decision output when AI violates policy
- What gets logged (identity chain, requested data, blocked data, reason)
- Human-readable explanation for compliance officers
- 42ms enforcement latency

**Key takeaway:** This is evidence, not logs. Audit-ready by design.

**Bonus:** Also open `prior_auth_decision_allow.json` to see a successful request.

---

### 3. **Understand the Rules** (3 minutes)

⚙️ Open: `policy_prior_authorization_v1.yaml`

**You'll see:**
- Permitted vs. prohibited PHI access (minimum necessary enforcement)
- Allowed vs. blocked actions (can draft, cannot submit without approval)
- Risk tier enforcement (different governance for different risk levels)
- Human approval triggers (high-cost, off-label, experimental)

**Key takeaway:** Policies are human-readable YAML. You control the rules.

---

### 4. **Know the Contract** (1 minute)

📐 Skim: `schemas/decision_schema.json`

**You'll understand:**
- Every Guardrail decision follows this schema
- Standardized format compliance teams rely on
- Exportable to CSV/JSON for auditors

**Key takeaway:** Consistent decision format = reliable audit trail.

---

### 5. **Try It (Optional)** (5-10 minutes if you want hands-on)

🚀 Follow: `/starter/README.md`

**You'll:**
- Validate the policy file
- Run sample requests through Guardrail
- Generate decision outputs
- Export compliance reports

**Key takeaway:** This isn't theoretical—it runs.

---

## What You Now Understand

After 10 minutes, you know:

✅ **What Guardrail is:** Independent policy enforcement for AI agents  
✅ **Where it sits:** Middleware between AI and protected resources  
✅ **How it works:** Intercept → Evaluate → Decide → Log  
✅ **What it produces:** Audit-ready decision objects  
✅ **Why it matters:** Hospitals can't deploy AI without provable compliance

---

## What To Do Next

### If You're Evaluating for Your Organization

**Next steps:**
1. Share `architecture/guardrail_overview.md` with your CTO
2. Share example decisions with compliance officer
3. Review policy file with security team
4. Contact us: healthcare-pilots@defiantindustries.com

### If You're Building on Guardrail

**Next steps:**
1. Read `architecture/data_flow.md` for technical deep-dive
2. Review `/mappings` for PHI taxonomy and action catalog
3. Customize `policy_prior_authorization_v1.yaml` for your needs
4. Check `/compliance/hipaa_mapping.md` for regulatory alignment

### If You're Exploring AI Governance

**Next steps:**
1. Read full README.md for use cases and ROI
2. Review PACKAGE_STRUCTURE.md to understand complete offering
3. Star/fork on GitHub (help us build the ecosystem)

---

## Three Audiences, Three Takeaways

**For CTOs:**
> "This is a policy engine for AI, not a wrapper around an API."

**For Compliance Officers:**
> "They can produce evidence, not just logs."

**For Investors:**
> "This is infrastructure with multiple verticals, not a healthcare app."

---

## Questions?

**Technical:** support@defiantindustries.com  
**Compliance:** compliance@defiantindustries.com  
**Pilot/Demo:** healthcare-pilots@defiantindustries.com

**GitHub:** [Repository URL]  
**Website:** defiantindustries.com

---

**You just completed the Golden Path.**

**Next:** Pick your audience path above and keep going.

---

**Version:** 1.0  
**Last Updated:** January 18, 2026
