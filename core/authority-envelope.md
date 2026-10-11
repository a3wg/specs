# A3 Authority Envelope

**Version:** 1.0.0
**Status:** Stable. Changes follow the versioning rules in the [README](../README.md#conventions).
**Licence:** Community Specification License 1.0 ([LICENSE.md](../LICENSE.md)). Independent implementation is explicitly permitted and encouraged.
**Depends on:** [Decision Trace Format](decision-trace.md) 1.0 (envelopes are recorded in and verified over a trace, and signed with its key set); [Capability Schema](capability-schema.md) 1.0 (clearance identifiers).

An agent acting for a person must never be able to do more than *both* of them are allowed to. Most systems get half of this: they check what the agent is allowed to do, or what the user is allowed to do, and then let the other half through on trust. The confused deputy — an agent used to exceed its caller's authority — lives in that gap.

This specification defines the **authority envelope**: a signed claim stating exactly what a task may do, computed as the **intersection** of the agent's grants and the originating principal's grants, scoped to one task, time-boxed, and carried across delegation so that a sub-task can never hold more than its parent. Because it is a signed object with both inputs inside it, any system that receives it — a tool gateway, another runtime, an auditor a year later — can check the intersection itself rather than trusting whoever handed it over.

The words MUST, SHOULD and MAY are used as in RFC 2119.

---

## 1. Properties

| Property | Meaning |
|---|---|
| **Intersection** | `clearances` is exactly what the agent holds ∩ what the principal holds. Never a union, never either side alone. |
| **Self-evidencing** | The envelope carries both sides (`held`), so the intersection is recomputable by anyone. |
| **Non-transferable** | It names one trace and one task; it is useless anywhere else. |
| **Monotone under delegation** | A child envelope's clearances, budget, expiry and depth are bounded by its parent's; its principal is its parent's. Authority flows down and only shrinks. |
| **Verifiable** | It is signed by the issuing control plane with a key from the trace's key set. |
| **Binding on actions** | Every action recorded in the trace names the clearances it required; a verifier checks each against the envelope. An approval authorises an action *within* the envelope; it never adds to it. |

---

## 2. The envelope

```json
{
  "v": "1.0",
  "trace_id": "019267b4-…",
  "task_id": "task-4471",
  "principal": {"type": "human", "id": "sub|8f3a"},
  "agent": {"name": "order-analyst", "namespace": "support"},
  "clearances": ["orders.read"],
  "held": {
    "agent": ["customers.read", "orders.read", "refunds.issue"],
    "principal": ["orders.read"]
  },
  "budget": {"tokens": 400000, "usd": "0.75", "wall_clock_ms": 600000, "actions": 40},
  "expires": "2026-09-16T14:10:00.000Z",
  "depth": 0,
  "parent": null,
  "issued_at": "2026-09-16T14:00:00.020Z",
  "issuer": "control-plane",
  "sig": "…"
}
```

| Field | Required | Meaning |
|---|---|---|
| `v` | yes | Specification version. |
| `trace_id`, `task_id` | yes | The one task this envelope is for. |
| `principal` | yes | The originating principal: `type` is `human` or `service`, never `agent`. |
| `agent` | yes | The agent definition that will act. |
| `clearances` | yes | **The intersection.** Sorted, unique. May be empty. |
| `held.agent` | yes | Every clearance the agent had been granted at issue. Sorted, unique. |
| `held.principal` | yes | Every clearance the identity interface reports the principal as holding at issue. Sorted, unique. |
| `budget` | yes | `tokens`, `usd` (decimal string), `wall_clock_ms`, `actions`. What the task may consume. |
| `expires` | yes | RFC 3339 UTC. Nothing may run under this envelope after it. |
| `depth` | yes | Delegation depth; 0 for a task submitted from outside. |
| `parent` | yes | `null`, or `{"trace_id", "seq"}` of the parent's `delegation.spawned` record. |
| `issued_at` | yes | RFC 3339 UTC. |
| `issuer` | yes | Key identifier of the signing control plane, resolvable in the trace's key set. |
| `sig` | yes | base64url Ed25519 signature over `utf8(envelope_hash)`. |

**`envelope_hash = sha256(JCS(envelope without sig))`**, encoded `sha256:<hex>`, using RFC 8785 canonicalisation exactly as the trace format does. Signing the hash string rather than the object keeps the construction identical to trace records, so the same keys and verifiers apply.

The envelope is recorded verbatim as `authority_envelope` in the trace's `task.admitted` record. The trace format requires only `clearances` and `expires` there; this specification requires the whole object.

---

## 3. Issuance rules

An issuer MUST:

1. Obtain `held.agent` from the agent's granted clearances and `held.principal` from the identity interface **at issue time**, not from any cached or caller-supplied value.
2. Compute `clearances` as the exact set intersection, sorted.
3. Set `expires` no later than the task's deadline, if it has one, and no later than the parent's `expires` for a delegated task.
4. Set `budget` no greater than the parent's remaining budget for a delegated task (§5).
5. Sign with a key whose key-set entry names the control-plane component.

An issuer MUST NOT issue an envelope whose `principal.type` is `agent`. An agent's own identity never originates authority.

---

## 4. Use rules

- A gateway or any other enforcement point MUST refuse an action whose required clearances are not a subset of `clearances`, and MUST record the required clearances on the action's `tool.invoked` record as `clearances_required`, so the check is verifiable.
- An action MUST NOT run after `expires`.
- An approval (Approval Protocol) authorises an action within the envelope. If the action's required clearances exceed the envelope, approval does not cure it; the action is refused before approval is sought.
- An enforcement point receiving an envelope from another system MUST verify its signature and scope before honouring it.

---

## 5. Delegation

When a task delegates work to another agent, the child task receives its own envelope, derived from the parent's. A child envelope MUST satisfy, relative to its parent:

| Rule | Requirement |
|---|---|
| **Principal** | `child.principal` equals `parent.principal`. Delegation never re-originates authority. |
| **Clearances** | `child.clearances` ⊆ `parent.clearances`. The child's `held.agent` is the child agent's own grants; `held.principal` is the parent's `clearances` (the parent's effective authority is the principal-side input to the child's intersection). |
| **Budget** | Each dimension of `child.budget` ≤ the parent's **remaining** budget at delegation: `parent.budget` minus the parent's recorded consumption and minus budgets already delegated to earlier children. |
| **Expiry** | `child.expires` ≤ `parent.expires`. |
| **Depth** | `child.depth` = `parent.depth` + 1, and ≤ the parent agent's declared maximum delegation depth. |
| **Link** | `child.parent` names the parent's `delegation.spawned` record; that record's `authority_envelope` equals the child envelope. |

A runtime MAY additionally require that a parent declare the kinds of work it delegates, and refuse a delegation outside that declaration. Such a refusal is a policy decision, recorded as one; it narrows what may be delegated and never widens it, so an envelope issued by a runtime that enforces it verifies exactly as one issued by a runtime that does not.

A dimension a parent never limited is written as zero and bounds nothing, so a parent that has spent a dimension it *did* limit MUST refuse the delegation rather than issue a child with zero on that dimension. Zero as "nothing left" and zero as "no limit" are indistinguishable in the record, and an implementation that hands the first down creates a child that is unbounded where its parent was exhausted.

The parent trace's `delegation.spawned` record carries the child envelope verbatim; the child trace's `task.admitted` carries the same object. A verifier with both traces checks them against each other.

---

## 6. Verification procedure

Runs over a trace, or a bundle of traces, that has passed the Decision Trace Format verifier. Every error is reported.

Per trace:

1. **Structure.** `task.admitted.authority_envelope` is a complete envelope. Else `E_STRUCTURE`.
2. **Scope.** `trace_id` and `task_id` equal the trace's. Else `E_ENVELOPE_SCOPE`.
3. **Signature.** `issuer` resolves in the key set to a control-plane key valid at `issued_at`, and `sig` verifies over `envelope_hash`. Else `E_ENVELOPE_SIG`.
4. **Intersection.** `clearances` equals sorted(`held.agent` ∩ `held.principal`). Else `E_INTERSECTION`.
5. **Principal.** `principal.type` is `human` or `service`, and equals `task.admitted.originating_principal`. Else `E_PRINCIPAL`.
6. **Expiry.** `expires` is not before `issued_at`; no `tool.invoked` has `ts` after `expires`. Else `E_ENVELOPE_EXPIRY`.
7. **Actions.** For every `tool.invoked`, `clearances_required` (absent means none) ⊆ `clearances`. Else `E_ACTION_UNAUTHORISED`. This applies equally to approval-gated invocations.

Per bundle, for every trace whose envelope has a non-null `parent`:

8. **Link.** The parent trace is in the bundle; the record at `parent.seq` is `delegation.spawned`; its `authority_envelope` equals this envelope (by `envelope_hash`). Else `E_CHILD_LINK`.
9. **Monotonicity.** Principal equal (`E_CHILD_PRINCIPAL`); clearances subset (`E_CHILD_WIDENED`); each budget dimension ≤ parent remaining at the spawn record (`E_CHILD_BUDGET`); expiry ≤ parent's (`E_CHILD_EXPIRY`); depth = parent + 1 (`E_CHILD_DEPTH`).

What the verifier cannot establish: whether `held.principal` was *true* — whether the identity interface reported honestly. It establishes that the issuer committed to a specific claim about it, under signature, and that everything downstream was consistent with that claim.

---

## 7. Relationship to the other specifications

- **Decision Trace Format** gives the envelope its record, its key set and its bundle linkage. The format's own bundle check (`E_ENVELOPE_WIDENED`) is a coarse version of §5; this specification is the precise, signed one.
- **Capability Schema** defines clearance identifiers and what an agent declares it holds. A declaration is what the agent *claims*; `held.agent` is what the issuer *confirmed* at issue.
- **Approval Protocol** operates within the envelope. The plan's `originating_principal` is the envelope's `principal`; a gated invocation is subject to rule 7 like any other.

---

## 8. Conformance

**Issuer:** produces envelopes that verify. **Enforcer:** refuses actions outside the envelope and records `clearances_required`. **Verifier:** produces the specified result for every vector in `vectors/authority-envelope/`. Initial vectors: a valid envelope with an action inside it; an intersection that includes something the principal does not hold; a bad signature; an action requiring a clearance outside the envelope; an approved action outside the envelope; a valid delegation bundle; a child with a clearance its parent lacks; a child budget above the parent's remaining; a child with a different principal; a child at the wrong depth; a child expiring after its parent; an envelope scoped to another task.

---

## 9. Settled in 1.0, and future work

Settled:

1. **`held.principal` records the principal, not the identity provider's assertion.** An optional `evidence` field carrying that assertion (for example an OpenID Connect token's issuer and `jti`) can be added in a minor version, so that a verifier can go one step further back.
2. **Budget across a delegation tree is "remaining at the spawn record"**, reconstructed from `budget.consumed` records. A parent that under-reports consumption can over-delegate. The control is the supervisor's signature on those records: the under-report is on the record, signed by whoever made it.
3. **An envelope cannot be narrowed by the submitter in 1.0.** A principal asking to run with less than they hold is useful, and can be added as an optional field in a minor version without changing how an envelope is computed when it is absent.
