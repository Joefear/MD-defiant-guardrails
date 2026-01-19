# Guardrail Healthcare Starter Pack v1.1 - File Structure

## Directory Layout

```
v1/
├── README.md                                    # 5-minute quick start guide
├── policy_prior_authorization_v1.yaml                              # Production-ready policy
├── TECHNICAL_IMPROVEMENTS_SUMMARY.md            # What fixes were applied
│
├── schemas/                                     # All schemas follow *_schema.* naming
│   ├── decision_schema.json                    # Canonical decision object format
│   ├── identity_chain_schema.yaml              # USER→ROLE→AGENT→POLICY audit trail
│   ├── risk_tier_schema.yaml                   # Risk-based enforcement profiles
│   └── decision_explainability_schema.yaml     # Human-readable output templates
│
├── examples/                                    # Real decision outputs
│   ├── prior_auth_decision_allow.json          # ALLOW decision (87ms)
│   ├── prior_auth_decision_block.json          # BLOCK decision (42ms)
│   └── prior_auth_decision_require_approval.json # REQUIRE_APPROVAL (156ms)
│
└── mappings/                                    # Foundation reference data
    ├── phi_tags.yaml                           # PHI classification taxonomy
    └── action_catalog.yaml                     # Healthcare actions + risk levels
```

## File Naming Convention

**Schemas:** `*_schema.{json|yaml}`
- Consistent naming makes it obvious what's a schema vs. implementation
- JSON for structured data formats (decision object)
- YAML for configuration/templates (identity chain, risk tiers, explainability)

**Examples:** `*_decision_{outcome}.json`
- Shows real decision output for each outcome type
- JSON format (standardized output)

**Policies:** `{workflow}_policy.yaml` or just `{workflow}.yaml`
- YAML for human-readable policy definitions
- Example: `policy_prior_authorization_v1.yaml`

**Mappings:** `{domain}_{type}.yaml`
- Reference data for policies to use
- Examples: `phi_tags.yaml`, `action_catalog.yaml`

## Quick Reference

### Core Files to Start With
1. `README.md` - Start here (5-min quick start)
2. `policy_prior_authorization_v1.yaml` - See production-ready policy
3. `examples/prior_auth_decision_allow.json` - See what success looks like

### Schema Files
- `decision_schema.json` - Every Guardrail decision follows this format
- `identity_chain_schema.yaml` - How audit trails are structured
- `risk_tier_schema.yaml` - How risk drives enforcement behavior
- `decision_explainability_schema.yaml` - How to generate OCR-ready reports

### Foundation Data
- `phi_tags.yaml` - Reference for all PHI fields + regulatory mappings
- `action_catalog.yaml` - Reference for all healthcare actions + risk levels

## Validation Checklist

✅ All schema files follow `*_schema.*` naming convention
✅ No inconsistent file extensions (.schema.json vs _schema.json)
✅ Documentation references use updated filenames
✅ No empty or erroneous directories
✅ Archive extracts cleanly with correct structure

---

**Version:** 1.1 (Standardized Naming)
**Last Updated:** January 18, 2026
**Package:** guardrail-healthcare-starter-v1.1.tar.gz

## Architecture Documentation

### Core Architecture
**`architecture/guardrail_overview.md`**
- What Guardrail is and where it sits
- Why independence matters
- Comparison to alternatives
- **Read this first for strategic understanding**

**`architecture/data_flow.md`**
- Detailed request/response flow
- Step-by-step policy evaluation
- Performance characteristics
- **Read this for technical deep-dive**

### When to Read What

**VCs/Investors:** Start with `architecture/guardrail_overview.md`
**CTOs/Architects:** Read both architecture docs
**Compliance/Legal:** Focus on decision examples + audit sections
**Implementation Teams:** Start with README.md, then data_flow.md

---

**Updated:** January 18, 2026 (Added architecture folder)
