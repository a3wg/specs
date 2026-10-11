# Bindings

A binding says how the core is carried in one environment or one protocol. An implementation in a given environment owes the core specifications and their vectors, plus a binding for its environment.

Whatever the environment, a binding has to establish four things:

1. **Instance identity.** How one running instance is told apart from every other. It binds to the instance, not to the class it was started from.
2. **Evidence of placement.** What evidence stands behind a claim about where an instance runs.
3. **Credentials.** Where a credential comes from, and how it is scoped to one task.
4. **The reference monitor.** The point a consequential action must pass through and cannot go around. In the institutions literature this is the *governor*. A binding that cannot name its reference monitor is not finished.

| Binding | Status |
|---|---|
| [Kubernetes](kubernetes.md) | 0.1 draft; no conformance suite yet |
| A2A | Planned: child envelope, trace link, key set and budget report in A2A task metadata |
| OAuth | Planned: plan binding in token exchange, and a rich-authorisation-request type for plans |
| MCP | Planned: side-effect classes and plan binding on MCP tool calls |
| OpenTelemetry | Planned: alignment with the GenAI semantic conventions |
| Edge devices | Planned: device identity, attestation, pre-issued credentials, a named local reference monitor |

A binding gets its own version, conformance vectors and at least one working implementation before it leaves draft.
