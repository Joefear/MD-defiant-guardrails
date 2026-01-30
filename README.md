Guardrail Decision Format (GDF)
An open, vendor-neutral standard for recording AI policy decisions and authorization outcomes across text, vision, voice, and action-intent systems.

GDF defines a minimal, extensible, machine- and human-readable event format for documenting how AI systems are governed by organizational policy, including:

Which system or model produced an output or action intent

Which policy snapshot was applied at the time of the decision

What the decision outcome was (allow, deny, escalate, modify)

What cryptographic evidence (hashes) supports the decision without requiring sensitive raw data by default

How related decisions are linked across systems and modalities within a single trace

GDF is designed to support:

Auditability and traceability for AI-enabled systems in enterprise and regulated environments

Interoperability across vendors, models, and governance platforms

Policy-based authorization workflows upstream of safety and execution systems

GDF does not define policy authoring languages, functional safety systems, or compliance certification methods. It provides a standardized record format that organizations can use as part of their broader governance, risk management, and compliance programs.
