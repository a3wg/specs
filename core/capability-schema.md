# A3 Capability Schema

**Version:** 1.0.0
**Status:** Stable. Changes follow the versioning rules in the [README](../README.md#conventions).
**Licence:** Community Specification License 1.0 ([LICENSE.md](../LICENSE.md)). Independent implementation is explicitly permitted and encouraged.
**Depends on:** [Decision Trace Format](decision-trace.md) 1.0 — placement decisions are recorded as trace records and verified over a trace.

Systems that place work usually choose by resources: processor, memory, the properties of a machine. For an agent that is the wrong question. The question is *can this agent do this, and is it allowed to?* This schema is a portable way to state both — what an agent **can** do (capabilities) and what it **may** do (clearances) — and to state what a task **requires**, so that any runtime can match one against the other and any verifier can check, from the trace alone, that a task was placed on an agent that satisfied every requirement.

The words MUST, SHOULD and MAY are used as in RFC 2119.

---

## 1. Design goals

| Goal | How the schema achieves it |
|---|---|
| **Portability** | Declarations and requirements are plain JSON documents with a fixed vocabulary; an agent card, a registry entry or a Kubernetes manifest can all carry one |
| **Explainability** | Every matching rule is a named **predicate**; a placement decision is recorded as the list of predicates that passed and, per rejected candidate, the one that failed |
| **Verifiability** | The trace carries the requirements and the chosen agent's declaration; a verifier re-runs the match |
| **Separation of can and may** | Capabilities and clearances are different things with different sources: competence is declared and may be tested; authority is granted and is checked against the grantor |

---

## 2. Identifiers

**Capability names** are lowercase, dot-separated: `<domain>.<competence>`, e.g. `order.investigate`, `text.summarise`. Each has an integer **major version**. Within a major version, behaviour is compatible; a new major version is a different capability for matching purposes.

**Clearance identifiers** are lowercase, dot-separated: `<domain>.<permission>`, e.g. `orders.read`, `refunds.issue`, `data.pii-read`. There is no implicit hierarchy: holding `refunds.issue` does not imply `orders.read`. A grantor that intends both grants both.

**Residency labels** are opaque strings an operator defines, e.g. `eu`, `us-gov`. They are matched by equality.

Grammar: `[a-z0-9]+(\.[a-z0-9-]+)+` for capability and clearance identifiers.

---

## 3. Declaration

What an agent declares about itself. Emitted by the runtime that hosts the agent; carried in the trace when the agent is chosen.

```json
{
  "v": "1.0",
  "agent": {"name": "order-analyst", "namespace": "support"},
  "capabilities": [
    {"name": "order.investigate", "version": 2},
    {"name": "case.summarise", "version": 1}
  ],
  "clearances": ["orders.read", "customers.read"],
  "residency": ["eu"],
  "model": {"latency_class": "interactive", "cost_ceiling_usd": "0.75"}
}
```

| Field | Required | Meaning |
|---|---|---|
| `v` | yes | Schema version. |
| `agent` | yes | `{name, namespace}`. `name` identifies the agent; `namespace` is the scope in which that name is unique, such as a tenant, a project or a team. The pair identifies the agent across every record that names it. |
| `capabilities` | yes | May be empty. Each `{name, version}`, optionally `description`, `inputs` and `outputs` (JSON Schema), `evaluation` (a reference to the probe that substantiates the claim). |
| `clearances` | yes | May be empty. Identifiers the agent has been **granted** — by a principal who held them, as the runtime's authority model requires. A declaration asserts the grant; it does not create it. |
| `residency` | no | Labels the agent's runtime satisfies. Absent means unconstrained. |
| `tools` | no | Names of the tools the agent is bound to, and so can actually call. A capability is what the agent knows how to do; a binding is what it can reach. |
| `model` | no | `latency_class` (`interactive` \| `batch`), `cost_ceiling_usd` (decimal string). |

A capability declaration is a **claim**. Whether it is substantiated — by an evaluation suite, by a registry that reviewed it, by nothing — is outside this schema; the `evaluation` field lets a runtime say how, and a scheduler MAY weight unsubstantiated claims lower.

---

## 4. Requirements

What a task requires. Carried in the trace in `task.submitted`.

```json
{
  "v": "1.0",
  "capabilities": [{"name": "order.investigate", "version": 2}],
  "clearances": ["orders.read"],
  "residency": "eu",
  "model": {"latency_class": "interactive"}
}
```

| Field | Required | Meaning |
|---|---|---|
| `capabilities` | no | Every listed capability must be declared at exactly that major version. |
| `clearances` | no | Every listed clearance must be held. |
| `tools` | no | Every listed tool must be bound to the agent. |
| `residency` | no | The agent's `residency` must contain this label. |
| `model.latency_class` | no | Must equal the agent's. |
| `session` | no | An identifier for a conversation this task continues; used for affinity, never for filtering. |

All fields absent means any agent is eligible.

---

## 5. Matching: predicates

Matching has two phases: **filter** (hard — an agent that fails any predicate is ineligible) then **score** (soft — among the eligible, which is best).

Only the filter is normative. Every filter rule is a **predicate** with a canonical string form, so that explanations are comparable across runtimes:

| Predicate | Satisfied when |
|---|---|
| `capability:<name>/<version>` | The declaration lists `<name>` at major version `<version>`. |
| `clearance:<id>` | The declaration lists `<id>`. |
| `tool:<name>` | The declaration's `tools` contains `<name>`. |
| `residency:<label>` | The declaration's `residency` contains `<label>`. |
| `model.latency_class:<class>` | The declaration's `model.latency_class` equals `<class>`. |
| `agentRef:<name>` | The task named this agent directly (direct binding); satisfied only by that agent. |
| `anti-affinity:<name>` | The task excludes this agent (e.g. a verification must not be done by the proposer); satisfied by every other agent. |

The predicate set for a requirements document is derived mechanically: one `capability:` predicate per required capability, one `clearance:` per required clearance, one `tool:` per required tool, one `residency:` if stated, one `model.latency_class:` if stated, one `agentRef:` if the task named an agent, one `anti-affinity:` per excluded agent (`exclude` in the requirements).

Scoring is left to the runtime. A runtime SHOULD consider, at minimum, **memory affinity** (an agent already holding the task's session), historical success rate for the task's kind, and cost — and MUST record whatever it used, per candidate, so that a reviewer can see why one eligible agent won over another.

---

## 6. Recording a placement

A placement is recorded in the trace format's `task.bound` record, whose `scheduling` object this schema defines:

| Field | Required | Meaning |
|---|---|---|
| `requirements` | yes | The requirements document of §4, as matched. |
| `declaration` | yes | The chosen agent's declaration of §3, as matched. |
| `candidates` | yes | Names of every agent considered. |
| `filters_passed` | yes | Every predicate the chosen agent satisfied. MUST equal the predicate set derived from `requirements`. |
| `filters_failed` | yes | `[{candidate, predicate}]` — for each ineligible candidate, the first predicate it failed. |
| `scores` | yes | `{candidate: {total, ...components}}` for each eligible candidate. |
| `route` | no | An identifier for the control-plane rule that added requirements the task did not ask for, if one did. |
| `timings` | no | What the placement cost the control plane: phase durations and the API-server list operations it took. Informational — see below. |

The `requirements` MUST also appear in `task.submitted`, so that what was asked and what was matched can be compared.

**A resumed placement.** A task that resumes after `task.preempted` or `task.parked` goes back to the agent it was admitted to, under the envelope it already has: no candidates are weighed, and repeating the first placement's lists would record a decision that was not made again. Its `task.bound`, which carries `resumed: true`, MAY therefore carry in place of `candidates`, `filters_passed`, `filters_failed` and `scores` a reference to the placement that weighed them:

| Field | Required | Meaning |
|---|---|---|
| `recorded_in` | yes | `{kind: "task.bound", seq}` — the earlier `task.bound` in the same trace whose explanation covers this placement. `seq` MAY be omitted only when the implementation cannot tell which; the trace must then hold at least one placement with a full explanation. |
| `requirements`, `declaration` | yes | As above: what this placement is held to. |
| `candidates_count`, `filters_passed_count`, `filters_failed_count`, `scores_count` | no | The sizes of the lists the referenced placement recorded. Informational. |

The lists are the part that grows with the fleet — about 100 KB a placement at a thousand candidates — and a task parked on approvals and sub-tasks is placed many times; this is what keeps it from being recorded many times.

`timings`, where present, carries `gather_ms`, `select_ms` and `launch_ms` — gathering the candidates, choosing between them, and reaching `Bound` from the decision — with `decided_at` as the instant the choice was made, and a `requests` object counting the list operations the placement performed, by resource.

It is **informational and is not verified**. Nothing a verifier checks depends on it, an implementation that omits it is conformant, and a reader must treat the numbers as the recording implementation's own account of its work rather than as an attested claim. It is specified here rather than left to each implementation because it answers a question that has no other answer: a placement that took a second could have spent it computing or waiting, the remedies are opposite, and the records are the only place both an operator and an outside reviewer can look. An implementation that records it MUST NOT let a figure it could not measure default to zero — the field is omitted instead, because a zero here reads as "instantaneous" and would make an unmeasured phase look like the cheap one.

A deployment may route a task through a rule that requires more than the task asked for: only agents bound to a particular tool, never the agent whose plan is under review. Requirements added that way MUST be recorded in `scheduling.requirements`, and `route` MUST name the rule that added them, so that a reviewer seeing a predicate the task never asked for can find out who did. A route may only **add**: a placement that drops a requirement the task stated is not conformant, whatever the route says.

An unschedulable task is recorded as `task.rejected` whose `reason` names at least one failed predicate and the candidates that failed it. "No agent available" is not a conformant reason.

---

## 7. Verification procedure

Runs over a trace that has passed the Decision Trace Format verifier. Every error is reported.

1. **Requirements.** From `task.submitted`, read `requirements` (absent means empty) and derive the predicate set S. From `scheduling.requirements` derive the predicate set P — what was actually matched, which equals S unless a route added to it.
2. **Placement.** On `task.bound`, read `scheduling`. Absent or malformed is `E_STRUCTURE`.
3. **Scope.** `scheduling.declaration.agent` MUST equal the `agent` in `task.admitted`. Else `E_DECLARATION_SCOPE`.
4. **Consistency.** S MUST be a subset of P: everything the task asked for was matched. If `scheduling.route` is absent, P MUST also equal S, since nothing is on record as having added anything. Else `E_REQUIREMENTS_MISMATCH`.
5. **Satisfaction.** Every predicate in P MUST be satisfied by `scheduling.declaration` per §5. Else, per predicate: `E_CAPABILITY_UNMET`, `E_CLEARANCE_UNMET`, `E_TOOL_UNMET`, `E_RESIDENCY_UNMET`, `E_MODEL_UNMET`, `E_AGENTREF_UNMET`, `E_ANTIAFFINITY_UNMET`.
6. **Explanation.** `filters_passed` MUST equal P as a set (`E_EXPLANATION`). The chosen agent MUST NOT appear in `filters_failed` (`E_EXPLANATION`). `scores` MUST contain the chosen agent (`E_EXPLANATION`). For a resumed placement that carries `recorded_in` (§6) instead: the record MUST carry `resumed: true`, `recorded_in.kind` MUST be `task.bound`, and `recorded_in.seq`, when present, MUST be the `seq` of an earlier `task.bound` in the trace that carried a full explanation — or, when absent, the trace MUST hold one. Else `E_EXPLANATION`. Steps 3–5 apply to it as to any placement; the explanation checks are those of the record it names.
7. **Rejection.** On `task.rejected` whose `reason` claims no eligible agent, the reason MUST contain at least one predicate string from §5. Else `E_REJECTION_REASON`.

What the verifier cannot establish: whether the declaration was *true* — whether the agent really can investigate an order, or was really granted `orders.read`. It establishes that the placement was consistent with what was declared, which is what a later dispute about placement is about.

---

## 8. Relationship to other schemas

- **A2A Agent Card.** An Agent Card's `skills[].id` maps to a capability `name`; the card has no version or clearance concept. A runtime MAY derive a declaration's `capabilities` from a card and MUST source `clearances` elsewhere.
- **MCP.** MCP describes tools, not agents; a tool listing is not a capability declaration. An agent that is bound to an MCP server may declare a capability that the server's tools make possible.
- **Authority Envelope** (planned). Clearances in a declaration are what the agent holds; the envelope is what a task may use — the intersection with the originating principal's grants. This schema describes the former only.

---

## 9. Conformance

**Declarer:** produces declarations and requirements that parse and use only identifiers of §2. **Scheduler:** records placements that verify. **Verifier:** produces the specified result for every vector in `vectors/capability-schema/`. Initial vectors: a valid placement; a capability not declared; a capability at the wrong major version; a clearance not held; a residency not satisfied; an explanation missing a required predicate; the chosen agent listed among the failed; a declaration for a different agent than the one admitted; a rejection whose reason names no predicate.

---

## 10. Example

```json
{"seq":1,"kind":"task.submitted","content":{"goal":"Investigate order A-1043","inputs":{"order":"A-1043"},"priority":"normal","deadline":"","idempotency_key":"…","requirements":{"v":"1.0","capabilities":[{"name":"order.investigate","version":2}],"clearances":["orders.read"],"residency":"eu"}}}
{"seq":2,"kind":"task.admitted","content":{"originating_principal":{"type":"human","id":"sub|8f3a"},"agent":{"name":"order-analyst","namespace":"support"},"authority_envelope":{"clearances":["orders.read","customers.read"],"expires":""},"budget":{…}}}
{"seq":3,"kind":"task.bound","content":{"instance":"task-4471-x","runtime":"node-a1","scheduling":{"requirements":{…as above…},"declaration":{"v":"1.0","agent":{"name":"order-analyst","namespace":"support"},"capabilities":[{"name":"order.investigate","version":2}],"clearances":["orders.read","customers.read"],"residency":["eu"]},"candidates":["order-analyst","summariser"],"filters_passed":["capability:order.investigate/2","clearance:orders.read","residency:eu"],"filters_failed":[{"candidate":"summariser","predicate":"capability:order.investigate/2"}],"scores":{"order-analyst":{"total":0.7,"affinity":0,"success_rate":1,"cost":0.4}}}}}
```

A reviewer reading record 3 knows why `summariser` was not chosen and why `order-analyst` was. A verifier confirms the declaration satisfies every predicate in `filters_passed`.

---

## 11. Settled in 1.0, and future work

Settled:

1. **Requirements name an exact major version.** That is what a verifier can check. Version ranges (`>=2`) can be added in a minor version without changing what an exact requirement means.
2. **A declaration's `clearances` are asserted, not proven.** Who granted a clearance is the subject of the Authority Envelope, where a future optional evidence field can carry it.
3. **`scores` are opaque.** A verifier checks that the chosen candidate passed every predicate and never compares scores. A normative minimum for score components can be added when there is a second runtime to compare against.
