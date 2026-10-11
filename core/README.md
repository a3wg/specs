# Core

The core specifications. They say nothing about where an agent runs, and they change rarely: a change here is a last resort ([GOVERNANCE.md](../GOVERNANCE.md), rule 3).

| Specification | Depends on |
|---|---|
| [Decision Trace Format](decision-trace.md) | none |
| [Approval Protocol](approval-protocol.md) | Decision Trace Format |
| [Capability Schema](capability-schema.md) | Decision Trace Format |
| [Authority Envelope](authority-envelope.md) | Decision Trace Format, Capability Schema |
| [Organisation Schema](organisation.md) | Capability Schema |
| [Key Discovery](key-discovery.md) | Decision Trace Format (§7) |

Four of the specifications describe what happened, and are verified over a trace that has already passed the Decision Trace Format's checks. The Organisation Schema describes an arrangement before anything runs. Key Discovery says where a verifier gets the keys all of them check signatures against.
