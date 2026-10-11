# A3 Kubernetes Binding

**Version:** 0.1 (draft)
**Status:** Draft. No conformance suite is specified yet.
**Licence:** Community Specification License 1.0 ([LICENSE.md](../LICENSE.md)). Independent implementation is explicitly permitted and encouraged.
**Depends on:** [Decision Trace Format](../core/decision-trace.md) 1.0, [Authority Envelope](../core/authority-envelope.md) 1.0, [Approval Protocol](../core/approval-protocol.md) 1.0, [Key Discovery](../core/key-discovery.md) 0.1.

The core specifications name an **instance**, a **runtime** and an **actor**, and never a pod or a cluster. This binding says what those are when agents run as pods on Kubernetes. It also sets out what an implementation there must establish, so that the core's records mean what they say.

Every binding answers the same four questions: how one instance is told apart from every other, what evidence stands behind a claim about where it runs, where its credentials come from, and where the reference monitor sits. Sections 2 to 5 answer them in that order.

This binding does not say how an implementation is built. Trace storage, policy engines, model providers, memory, inventory and notification are each an implementation's own business, and nothing here requires a particular API for any of them.

The words MUST, SHOULD and MAY are used as in RFC 2119.

---

## 1. Terms

| Term | In this binding |
|---|---|
| **Installation** | The components that sign records for one control plane, with one key-set document under [Key Discovery](../core/key-discovery.md). |
| **Instance** | One pod running one agent for one task. `task.bound.content.instance` names it. |
| **Runtime** | The node the instance's pod is scheduled on. `task.bound.content.runtime` names it. |
| **Component** | A process that signs records: the control plane, a per-node component, the reference monitor. |
| **Reference monitor** | The component every consequential action passes through (§5). The core specifications call it the gateway. In the institutions literature, it is the *governor*. |

---

## 2. Instance identity

Every component an instance talks to answers one question first: is this request really from instance pod P in namespace N? Everything downstream depends on the answer: the authority envelope, the budget, the clearances, and the trace the action is written to. The pod is what binds a running agent to the task it was created for. A wrong answer here is the same failure as a wrong policy decision, made earlier.

**A claim.** An identity source turns a request into `{namespace, pod, source, subject}`: the instance it established, how it established it, and the identity document's own name (a SPIFFE ID or a certificate name). For a bearer token the name is empty, because a token names nothing.

**Three outcomes.** A source reports one of:

- a **claim**: this request is from that instance;
- **none**: no credential of this source's kind was presented;
- a **refusal**: something was presented, and it is wrong.

An implementation MUST NOT report *none* when it means *refusal*. A source that answers "nothing was presented" about a document it found invalid passes the decision to whatever runs next, and an installation with any other source configured then accepts the forgery. A document from an unexpected trust domain, a path that does not fit, or a signature that does not verify is a refusal.

**Combining sources.** A component MAY check several sources:

- Every source that reports a claim MUST name the same instance. "Any one is enough" makes the weakest credential the real one, because the attacker chooses which to present.
- Every source the installation marks as required MUST report a claim.

**An implementation MUST:**

- **Bind to the pod, not to the workload class.** A SPIFFE ID naming a service account identifies every pod that runs under it, so any of them could speak for any other. Instances of the same agent are exactly the population an attacker is already inside. The identity MUST distinguish one pod from another.
- **Fail closed.** A component with no identity source configured MUST refuse every request from an instance.
- **Bind the credential to the connection where it can.** A credential bound to the transport, such as a client certificate or an X.509 SVID, SHOULD be required wherever one is available. A bearer token cannot be bound to the connection it arrived on, which makes it the weakest source. It SHOULD NOT be the only required one in an installation that has anything better.

---

## 3. Placement evidence

A task's records say where it ran, and policy gates on that: an agent declaring `residency: eu` ([Capability Schema](../core/capability-schema.md)) must not be placed on a node that does not satisfy it. A claim with that much weight needs to show who is making it. A node label is an administrator's assertion, and nothing in a trace should present it as more than that.

**Evidence, not claims.** Evidence about a node has this shape:

```
{node, residency, isolation_class, extra, attestor, format, verified, at, measurements, statement, signature?}
```

- `attestor` names what established the evidence.
- `format` names its kind. `assertion` means there is nothing behind it: a label, a configuration value. Other formats name a checkable statement, such as `tpm2-quote`.
- `verified` is what the **receiving** side concluded. It is never what the sender said.

**A verifier of placement evidence MUST:**

- **Issue the challenge itself.** The verifying side chooses a fresh challenge for each attestation. A service that chooses its own nonce can replay a quote taken when the machine was in a state the installation would have accepted.
- **Ignore `verified` on the wire.** Evidence that says it has been verified has verified nothing. The verifier MUST drop whatever arrives in that field.
- **Refuse a format it cannot check.** Passing unknown evidence through as though it were understood puts an attestor's name on the record without the checking that the name implies.
- **Refuse an unenrolled machine.** Which key speaks for which machine is decided per machine. Trusting the key that arrives with the evidence makes the evidence prove nothing. One shared bundle for all nodes lets any enrolled node attest as any other.
- **Treat an unreachable attestor as an error.** "We could not ask" and "it claims nothing" are different answers, and only one of them should end a task.

**For `tpm2-quote`.** `statement` is the `TPMS_ATTEST` structure, and `signature` is over its SHA-256 digest. A verifier MUST check that:
- the structure carries `TPM_GENERATED_VALUE`;
- the signature verifies against the machine's enrolled key;
- every register the installation requires is covered;
- the register values match what the installation expects.

A quote proves how a machine booted. On its own it says nothing about where the machine is or what else runs on it, so **the claims MUST be bound into the quote**. The qualifying data is `SHA-256(challenge || SHA-256(canonical claims))`, so the TPM's signature covers the claims as well as the registers. Without that binding, a genuine quote can be presented beside any claims at all.

An installation MAY run on assertions alone. If it does, it records them with `format: assertion` and `verified: false`, and never presents them as anything stronger.

---

## 4. Credentials

An instance reaches tools with credentials it does not hold. The reference monitor (§5) obtains a credential for one tool call, presents it to the tool, and gives it up when the call ends. The agent never sees it.

**A credential lease:**

- MUST be scoped to one tool call, and released when the call ends;
- MUST NOT appear in any record or in any agent-readable memory ([Decision Trace Format](../core/decision-trace.md) §11). A record MAY carry the lease's identifier, as `tool.invoked.credential_lease_id`;
- MUST NOT be granted beyond what the store backing it will honour. A lease that claims a longer life than its credential has turns into a call that fails halfway through, for a reason nobody traces back to the lease.

**Revocation is reported honestly.** Releasing a lease over a *stored* secret forgets a mapping. The credential itself keeps working, and whatever read it during the call keeps it for as long as it is valid. Releasing a lease over a *minted* credential, such as a database user created for this lease or a short-lived cloud role, destroys it at the source. Both are useful, and they are different promises. Whatever issues a lease MUST report which one it issued (`revocable: true` only for the second), and MUST NOT claim the stronger one. When it does not know, it reports `revocable: false`. A failed revocation MUST be reported as a failure, even when the local lease is gone.

**Issuance fails closed.** If no credential can be obtained, the tool call is refused. It is never made without the credential it was supposed to carry. A component that brokers credentials MUST NOT log or echo them, including in error messages.

---

## 5. The reference monitor

An agent's consequential actions MUST pass through a reference monitor that the agent cannot go around. The monitor:
- enforces the authority envelope ([Authority Envelope](../core/authority-envelope.md) §4);
- recomputes plan hashes and enforces approvals ([Approval Protocol](../core/approval-protocol.md) §4);
- obtains credentials (§4);
- signs the `tool.invoked` and `tool.completed` records.

A pod's only reach is the network, so on Kubernetes the reference monitor is a network service, and "cannot go around" is a property of the network.

**An installation MUST:**

- **Confine the instance's egress.** An instance pod's network egress MUST be limited, for example by a NetworkPolicy, to the reference monitor and the installation's own components. An instance that can open a connection to a tool directly has a path around every rule in the core.
- **Keep credentials and signing keys out of the instance.** The pod MUST NOT hold a credential for any tool, or a signing key whose records claim to come from a component other than the instance's own actor. Where the platform would mount a credential by default, such as a service account token, the installation MUST disable it, or MUST state that the instance's actor is the holder of that account.
- **Not let an action choose its own executor.** Where a tool is run as a workload rather than called as a service, the tool's definition MUST NOT choose the image or the pod template. It refers to a template an administrator created, and the template MUST carry the annotation `a3wg.org/tool-invocable`. A template that exists only to run on a schedule has not been consented to as a tool. Running it with arguments an agent supplies is a confused deputy. The annotation's value is `true`, or `with-service-account-token` when the template's pod would be given a service account token, including when the template does not say and the platform would mount one by default. A reference monitor MUST refuse a template without the annotation, and MUST refuse one that would receive a token unless the value is `with-service-account-token`.

**Calls through the monitor:**

- A call MUST carry an idempotency key. A retry with the same key MUST NOT repeat a side effect.
- A call that waits on an approval or a long-running workload MUST be bounded in time. A call that times out MUST fail, never be left running unrecorded.

---

## 6. Keys and signing

- Each component that signs publishes its keys through the installation's key-set document ([Key Discovery](../core/key-discovery.md)).
- **Startup check.** Before writing a record, a component MUST confirm that the key it will sign with is the one the key-set document publishes for its `kid`, and refuse to start otherwise. A component with the wrong key writes records that verify nowhere, and without this check nobody finds out until someone reads one.
- A private signing key SHOULD NOT be readable by anything that can read Secrets in the component's namespace. Anyone who can read the key can write records in that component's name, which makes every trace it signed deniable. Holding keys in a separate signing service, a KMS or a hardware module meets this.

---

## 7. Conformance

One conformance role: **Installation**. A deployment of an implementation on Kubernetes satisfies §2 to §6.

Most of these obligations cannot be checked from a trace. They are properties of a running installation: what a pod can reach, what a verifier refuses. That is what a binding is for, but it means conformance has to be shown by probing a live installation with a black-box suite, and not by vectors. The suite is not specified yet. Until it is, a conformance claim against this binding is the implementer's own statement.

---

## 8. Open questions

- **Placement evidence in the trace.** The Decision Trace Format lets `task.bound` carry `runtime_attestation` (§4.1). This binding does not yet say when an installation MUST record it, or how a verifier tells attested placement from asserted placement after the fact.
- **Revocability in the trace.** Whether a credential was actually destroyed (`revocable` on a lease, §4) is an outcome worth recording. Recording it would be an optional content field, added in a minor version of the Decision Trace Format.
- **The suite.** A black-box conformance suite for §2, §3 and §5 is the next piece of work. It is what this binding needs to reach 1.0.
