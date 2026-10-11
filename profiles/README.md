# Profiles

A profile adds optional constraints or rules to the core for one need. An implementation can conform to the core without any profile. A profile never loosens a core rule.

"Profile" means only this. The part an implementation plays, such as Emitter, Verifier or Requester/Executor, is a **conformance role**, and roles are defined in each specification's conformance section.

None is published yet. Planned, in rough order:

| Profile | What it adds |
|---|---|
| Trace anchoring | External anchoring of traces and key-set history in a transparency log (SCITT), and RFC 3161 timestamps |
| Regulatory mappings | Testable statements that a trace conformant to the profile carries the records a named control asks for (EU AI Act, ISO/IEC 42001, NIST AI RMF and others). Evidence, not certification |
| Plan-class authorisation | A closed vocabulary of bounds, so that one approval can cover a class of plans |
| Compact | CBOR/COSE encoding, checkpoints and explicit loss records for constrained devices |

A profile enters when an adopter needs it, with vectors.
