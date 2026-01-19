Guardrail Decision Format (GDF) — Partner / Buyer Summary (v1.0)

What this is
GDF is the standard decision artifact produced by Guardrail when an AI system attempts to access sensitive data or take an action (send, submit, change, approve, message, etc.).
It’s designed to be the record a compliance officer, security team, or auditor can rely on—not a vague “log line.”

What it solves (in plain language)
When AI systems act inside regulated workflows, the core problem isn’t “accuracy.”
It’s: Who asked the AI to do something, what did it try to do, what rules applied, what was allowed/blocked, and what evidence proves it—after the fact.

GDF turns that into a repeatable, portable, machine-readable artifact.

What a partner gets immediately

Audit-ready proof of enforcement (what was requested, what was released, what was blocked)

Identity chain (USER → ROLE → AGENT → SYSTEM) so the question “who authorized this?” is always answerable

Reason codes + rule matches so compliance/security can trend, alert, and investigate

Optional integrity controls (hash, signatures, chaining) so records can be made tamper-evident

Tracing fields so this drops into enterprise observability stacks (trace/span/request IDs)

Why this matters commercially

Buyers don’t just want “guardrails.” They want evidence.

GDF is a contract between:

AI builders (who need predictable integration),

compliance/security (who need enforceable controls),

and executives (who need defensible risk posture).

This creates a platform-style advantage: once a customer standardizes on GDF, Guardrail becomes the system-of-record for AI governance events.

Where it fits
Guardrail sits between:

the AI agent/model/tooling layer (LLM, agent framework, MCP tools)

and regulated systems (EHR, claims, finance systems, customer data, workflows)

GDF is what Guardrail outputs every time it makes an enforcement decision.

Minimum required fields (what’s always captured)

Decision + timestamp

Policy ID/version (and optional hash)

Workflow/purpose

Identity chain

Requested data/actions (even if empty)

Risk tier (and optional score/factors)

Optional but high-credibility fields

Matched rules

Notifications sent (compliance/security)

Integrity signature / hash chaining

Evidence references (for fact-check / hallucination controls)

Domain extensions (healthcare/finance/etc.)

Why this is partner-friendly

Vendor-neutral structure

Extensible without breaking core compatibility

Works as a shared interface between Guardrail and:

SIEM / SOAR

ticketing (ServiceNow/Jira)

compliance review workflows

observability tracing systems