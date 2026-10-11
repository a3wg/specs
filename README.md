# Accountable Agent Action (A3)

The specifications of the **A3 Working Group** ([a3wg.org](https://a3wg.org)). A3 lets any agent, in any runtime, on any device:

- prove what authority it acted under,
- show that consequential actions were approved before they happened, and
- leave a record that someone who was not there can verify afterwards.

A3 specifies what can be checked from outside the agents: authority, approval, placement, the arrangement of agents and tools, and the record of what happened. It does not specify policy languages, orchestration or how a model behaves.

A3 is independent. It is not related to Google's or the Linux Foundation's A2A protocol, which it plans to bind to.

Each specification can be implemented from its text alone and is checked by normative test vectors. No particular runtime is required. The **reference implementation** is [Rashnu](https://rashnu.dev). It generates the vectors and shows that the specifications can be implemented. It is not normative: where Rashnu and the text disagree, the text wins and Rashnu is wrong. Every other runtime that emits conformant records is equally welcome, and the reference role is not permanent.

## The specifications

### Core

The core says nothing about where an agent runs. It names an instance, a runtime and an actor, never a pod, a container or a cluster.

| Specification | File | Version | Status |
|---|---|---|---|
| Decision Trace Format | [core/decision-trace.md](core/decision-trace.md) | 1.0.0 | Stable; 19 vectors |
| Approval Protocol | [core/approval-protocol.md](core/approval-protocol.md) | 1.0.0 | Stable; 16 vectors |
| Capability Schema | [core/capability-schema.md](core/capability-schema.md) | 1.0.0 | Stable; 18 vectors |
| Authority Envelope | [core/authority-envelope.md](core/authority-envelope.md) | 1.0.0 | Stable; 12 vectors |
| Organisation Schema | [core/organisation.md](core/organisation.md) | 1.0.0 | Stable; 21 vectors |
| Key Discovery | [core/key-discovery.md](core/key-discovery.md) | 0.1 | Draft; no vectors yet |

### Bindings

A binding says how the core is carried in one environment or protocol. For example, it says how one running instance is told apart from every other, and where the reference monitor sits: the point every consequential action must pass through.

| Binding | File | Version | Status |
|---|---|---|---|
| Kubernetes | [bindings/kubernetes.md](bindings/kubernetes.md) | 0.1 | Draft; no conformance suite yet |

Planned: A2A, OAuth (token exchange and rich authorisation requests), MCP, OpenTelemetry, edge devices. See [bindings/README.md](bindings/README.md).

### Profiles

A profile adds constraints or extra rules for one need, and is optional. None is published yet. Planned profiles cover trace anchoring, compact encoding, plan-class authorisation and regulatory mappings. See [profiles/README.md](profiles/README.md).

### Test vectors

[vectors/](vectors/) holds the normative vectors, one directory per specification. A conformant implementation produces the stated result for every vector.

## Conventions

- Requirement words **MUST**, **SHOULD** and **MAY** follow RFC 2119.
- Each specification, binding and profile has its own version, `MAJOR.MINOR.PATCH`:
  - **PATCH** is editorial: clarifications, typos, examples. Nothing an implementation does changes.
  - **MINOR** adds without breaking: a new record kind, an optional content field, a new error code for a case that was previously unspecified. A conformant 1.0 implementation stays conformant.
  - **MAJOR** breaks: a change to header fields, hashing, signing or a verification rule that changes the result for an existing input.
- Records and documents carry `MAJOR.MINOR` (for example `"v": "1.0"`, `"format": "a3-trace/1.0"`). A verifier accepts every minor version of the major versions it implements.
- A specification marked **Draft** is versioned `0.x` and can change in any way.
- Every change to the wire format or the verification rules ships with vectors.
- A conformance claim names what it conforms to: core specification, binding and profile, each with a version and a role. For example: "Decision Trace Format 1.0, Verifier role".
- "Role" means the part an implementation plays, such as Emitter, Verifier or Requester/Executor. "Profile" means only an optional set of constraints for a specific need, as above.

## Repository

| File | What it is |
|---|---|
| [LICENSE.md](LICENSE.md) | Community Specification License 1.0 for the specifications; Apache 2.0 for code |
| [SCOPE.md](SCOPE.md) | What this working group covers. It also sets the bounds of the patent commitment |
| [NOTICES.md](NOTICES.md) | Contacts, licence acceptances, exclusions, names and marks |
| [GOVERNANCE.md](GOVERNANCE.md) | Who decides, and how |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to propose a change |
| [CONTRIBUTOR-LICENSE-AGREEMENT.md](CONTRIBUTOR-LICENSE-AGREEMENT.md) | What a contributor agrees to |
| [SECURITY.md](SECURITY.md) | How to report a flaw in a specification |

## Contact

[info@a3wg.org](mailto:info@a3wg.org) for anything about the Working Group. Changes to the specifications go through [issues and pull requests](https://github.com/a3wg/specs).
