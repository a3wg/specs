# A3 Organisation Schema

**Version:** 1.0.0
**Status:** Stable. Changes follow the versioning rules in the [README](../README.md#conventions).
**Licence:** Community Specification License 1.0 ([LICENSE.md](../LICENSE.md)). Independent implementation is explicitly permitted and encouraged.
**Depends on:** [Capability Schema](capability-schema.md) 1.0 — roles are declarations and entry points are requirements, matched by that schema's predicates.

The four specifications this one joins are all about a piece of work that has happened or is happening. The [Decision Trace Format](decision-trace.md) is the record of what an agent did, the [Approval Protocol](approval-protocol.md) is what a human agreed to, the [Capability Schema](capability-schema.md) is what one agent declares and how a task was placed on it, and the [Authority Envelope](authority-envelope.md) is what one task may do while it runs.

This one is about the **arrangement** those happen inside, before any of them do: which roles exist, what each may do and reach, who may ask whom for help, which actions a human has to sign off, and how work enters at all. It is a portable document, and it is checkable: an arrangement can be incoherent in ways nobody notices until the day it matters, and most of those ways can be found by reading the document alone.

Nothing here reads an agent's code. A role states what it may ask for in the same way it states what it can do, and the runtime holds it to the statement. What this schema verifies is the **set of runs an arrangement permits**, not the run any particular agent will choose.

The words MUST, SHOULD and MAY are used as in RFC 2119.

---

## 1. Design goals

| Goal | How the schema achieves it |
|---|---|
| **Code independence** | Every edge is declared, never inferred from an implementation; a role written in any language, or shipped as a binary, is described the same way |
| **Portability** | One JSON document describes an arrangement, whatever runtime hosts it |
| **Checkability** | Coherence is a fixed list of named rules over the document, each producing a stable code |
| **Enforceability** | Everything the document declares is something the runtime can refuse at the moment it is exceeded, so a stale document fails loudly rather than misleading quietly |

---

## 2. Identifiers

Role, tool, store and gate names are the names those objects have in the deployment that produced the document. Capability and clearance identifiers follow the [Capability Schema](capability-schema.md) §2. Budgets and durations are integers; currency is a decimal string, as in the [Authority Envelope](authority-envelope.md).

---

## 3. The document

```json
{
  "v": "1.0",
  "organisation": {"name": "globex"},
  "roles": [...],
  "tools": [...],
  "stores": [...],
  "gates": [...],
  "entries": [...],
  "principals": [...]
}
```

| Field | Required | Meaning |
|---|---|---|
| `v` | yes | Schema version. |
| `organisation` | yes | `{name}`: what this arrangement is called. A namespace, a tenant, a team. |
| `roles` | yes | The agents, §3.1. |
| `tools` | yes | What the roles may reach, §3.2. |
| `stores` | no | Shared memory, §3.3. |
| `gates` | no | Which actions need a human, §3.4. |
| `entries` | no | How work arrives, §3.5. |
| `principals` | no | The people the work runs for, §3.6. |

### 3.1 Roles

```json
{
  "name": "support-lead",
  "capabilities": [{"name": "case.triage", "version": 1}],
  "clearances": ["orders.read", "refunds.issue"],
  "tools": [{"name": "refund-processor", "mode": "invoke"}],
  "memory": [{"store": "case-log", "mode": "read"}],
  "delegation": {
    "max_depth": 1, "max_per_task": 3,
    "kinds": [{"name": "investigate-order", "description": "Find out what went wrong, from the order records.",
               "capabilities": [{"name": "order.investigate", "version": 1}], "tools": ["order-lookup"]}]
  },
  "limits": {"tool_calls_per_task": 0,
             "budget": {"tokens": 25000, "usd": "0.50", "wall_clock_ms": 1500000, "actions": 6}}
}
```

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | The role's name. |
| `capabilities` | yes | What it can do. May be empty. |
| `clearances` | yes | What it may authorise. May be empty. |
| `tools` | no | `{name, mode}` per binding; `mode` is `read` or `invoke`. A role can reach nothing it does not list. |
| `memory` | no | `{store, mode}` per binding; `mode` is `read`, `write` or `read-write`. |
| `delegation` | no | What this role may hand to another, and how much: `max_depth` (how deep a chain starting here may go; 0 forbids delegating), `max_per_task` (sub-tasks one task may start), and either `kinds` or `"unconstrained": true`. **Absent means unconstrained**, which is honest about a deployment that has not declared its intentions and is why an arrangement with no `delegation` anywhere can still be checked, only less deeply. A producer that knows a role may not delegate says so: `{"max_depth": 0, "max_per_task": 0}`. |
| `delegation.kinds` | no | The kinds of work this role may hand on. Each has a `name`, unique within the role; a `description`, written for whoever chooses among them, person or model; and `capabilities`, `clearances` and `tools`, which together are a requirements document of the Capability Schema §4. A role with `delegation` present, no `kinds` and no `unconstrained` may hand on nothing. |
| `limits` | no | Tool calls per task, and the per-task budget. |
| `suspended` | no | The role takes no new work. |

A role is a Capability Schema **declaration** with the extra fields above. A requirements document is satisfied by a role exactly when it is satisfied by that declaration, by that schema's predicates and no others.

### 3.2 Tools

```json
{"name": "refund-processor", "side_effect": "mutating", "requires_clearances": ["refunds.issue"],
 "operations": [{"name": "issue-refund", "side_effect": "mutating", "dry_run": true}]}
```

`side_effect` is the classification the deployment gives the tool: `read`, `mutating`, `destructive`, `irreversible`, `financial`, `communicating-externally` or `physical`. An operation inherits the tool's classification unless it states its own. `dry_run` says the operation can report what it would do without doing it.

The vocabulary runs along two axes and is not ordered. `mutating`, `destructive` and `irreversible` say how far an action can be walked back; `financial`, `communicating-externally` and `physical` say what it touches. A checker MUST treat every value other than `read` as consequential, and MUST NOT infer a ranking between them: an implementation that enumerates the consequential values stops covering whatever this list gains next, which is the failure mode this note exists to prevent.

### 3.3 Stores

`{"name": "case-log", "scope": "tenant"}`, where `scope` is `session`, `agent`, `tenant` or `shared`.

### 3.4 Gates

```json
{"name": "refund-approvals",
 "matches": {"side_effect": ["mutating", "destructive"], "tools": ["refund-processor"]},
 "approvers": {"groups": ["refund-leads"], "quorum": 1},
 "verifier": {"capabilities": [{"name": "action.verify", "version": 1}], "anti_affinity": true},
 "timeout_ms": 600000, "on_timeout": "deny"}
```

A gate is an approval rule, in the terms of the [Approval Protocol](approval-protocol.md). It matches actions by side effect, by tool, or both; it names who may decide and how many of them; it may require a reviewing role first; and it says what happens when nobody decides in time.

### 3.5 Entries

```json
{"name": "case-intake",
 "steps": [{"name": "triage", "requirements": {"v": "1.0", "capabilities": [{"name": "case.triage", "version": 1}]}}],
 "on_exhausted": "escalate"}
```

How work arrives and what it is matched against. Each step's `requirements` is a Capability Schema requirements document. `on_exhausted` is `reject`, `queue` or `escalate`.

### 3.6 Principals

`{"id": "sub|support-agent", "type": "human", "clearances": ["orders.read"], "groups": ["refund-leads"]}`

Who the work runs for. A deployment that establishes principals from an identity provider MAY emit an empty list, and the checks that depend on principals then report nothing rather than guessing.

---

## 4. Deriving the document

A runtime produces this document from whatever it uses to configure itself. The reference implementation, Rashnu, derives it from Kubernetes resources: an `Agent` becomes a role, a `Tool` a tool, a `MemoryStore` a store, an `ApprovalPolicy` a gate, a `TaskRoute` an entry, a `PrincipalGrant` a principal.

A document MUST describe one arrangement completely. Checking a document that omits roles reachable from its own entries produces findings about the omission, not about the arrangement.

---

## 5. Coherence

An arrangement is **coherent** when none of the rules below produces an `error`. Warnings and notes do not make it incoherent; they are the things worth looking at before they become cases.

Every rule names the subject it is about, so a finding can be acted on without reading the whole document.

| Code | Severity | The arrangement contains |
|---|---|---|
| `C_UNGUARDED_ACTION` | error | A role bound in `invoke` mode to a tool or operation whose side effect is not `read`, with no gate matching it. Something consequential can happen that no human is ever asked about. |
| `C_NO_APPROVER` | error | A gate whose approver groups are empty, or whose quorum exceeds the number of principals known to hold those groups. Every matching action waits for a decision that cannot arrive. |
| `C_NO_VERIFIER` | error | A gate requiring a reviewing role whose capability no role declares; or, with anti-affinity, where the only role declaring it is the only role that could propose the action it would review. |
| `C_UNSATISFIABLE_ENTRY` | error | An entry step whose requirements no role satisfies. Work of that kind arrives and can never be placed. |
| `C_UNSATISFIABLE_INTENT` | error | A declared delegation no role satisfies. A role may ask for work nobody in the arrangement can do. |
| `C_AUTHORITY_GAP` | error | A declared delegation requiring a clearance the delegating role does not hold. Authority only narrows as work passes down, so the delegation can never succeed. |
| `C_DELEGATION_DISABLED` | error | A role that declares kinds of delegation, or unconstrained delegation, while its `max_depth` or `max_per_task` forbids delegating at all. |
| `C_TOOL_CLEARANCE_UNHELD` | error | A tool requiring a clearance that no role bound to it holds. The call is refused at the gateway every time. |
| `C_TIME_SHORTFALL` | warning | A role that can invoke a gated tool with a per-task wall-clock budget shorter than the gate's timeout. On a runtime that keeps the waiting task on its own clock, the task dies while the approver is still deciding; one that parks it and pauses the clock loses nothing. |
| `C_UNREACHABLE_ROLE` | warning | A role no entry and no declared delegation can reach. |
| `C_ORPHAN_TOOL` | warning | A tool no role is bound to. |
| `C_STORE_NO_WRITER` | warning | A store no role may write. |
| `C_STORE_NO_READER` | warning | A store no role may read. |
| `C_NO_DRY_RUN` | note | A gated operation that cannot say what it would do, so approvers decide without seeing the consequence. |
| `C_DELEGATION_UNDESCRIBED` | note | A kind of delegation with no description. Whoever chooses among a role's kinds — increasingly a model — chooses blind. The finding names the role as its subject and the kind in its detail. |

Two rules deserve their reasoning stated, because they are the ones that look like false positives and are not.

**`C_AUTHORITY_GAP`.** A child's authority is its parent's authority intersected with its own. A role that delegates work needing `refunds.issue` while holding only `orders.read` hands down nothing it did not have, so the child is admitted and refused at its first consequential action. The delegation is not merely unwise; it cannot ever work.

**`C_TIME_SHORTFALL`.** An approval is a human waiting. A runtime that keeps the proposing task on its own clock while it waits has decided in advance, with a per-task budget shorter than the approval timeout, that no human will ever be fast enough. A runtime that parks the waiting task and pauses its clock, as the reference implementation does, loses nothing to it. The finding is therefore a warning rather than an error: it names an arrangement that only works on one kind of runtime, which a portable document should not assume.

---

## 6. Properties over paths

The rules of §5 are local: each looks at one role, one gate, one tool. Some things are only true of a **chain**. Work arrives at one role, which hands part of it to another, which hands part of that on again, and three of the runtime's rules compound along the way:

- **authority narrows at every hop**, so a clearance the first role holds may be gone before the work reaches the role that needs it;
- **a child's budget is carved from its parent's**, so the time available at the end of a chain is the least anyone along it allows;
- **each hop is bounded by the delegating role's depth limit**, so a declared chain can have a last hop that never happens.

A checker walks the chains the arrangement permits, starting at the roles an entry can place work on and following each role's `delegation.kinds`, and applies the properties below. A role whose `delegation` forbids delegating has no outgoing chains; a role with no `delegation` is unconstrained, and its depth unbounded.

| Code | Severity | Along some chain |
|---|---|---|
| `P_TIME_DEAD_END` | error | A role reaches a gated action with less time than the gate allows a human, although its own budget would have been enough. Some role earlier in the chain is the constraint. |
| `P_AUTHORITY_DEAD_END` | warning | A role reaches an action needing a clearance that the chain has already narrowed away. That route can never act, though another route might. |
| `P_DEPTH_EXCEEDED` | warning | A role declares a kind of delegation that, at the depth work reaches it, its own `max_depth` forbids. |
| `P_CYCLE` | warning | A role may delegate to a role already in the chain. Work that goes round runs out of depth instead of finishing. |
| `P_UNREACHABLE_ACTION` | note | A role may invoke something consequential, and no chain from any entry reaches that role. |

A finding from this section MUST carry the chain it was found on, as an ordered list of role names from the entry to the subject. The chain is not part of conformance; it is what makes the finding answerable rather than merely true.

A checker MAY bound the walk. One that stops early MUST say so rather than report a partial result as complete.

---

## 7. Verification procedure

1. **Parse.** A document that is not well formed is `E_STRUCTURE`, and nothing else is reported.
2. **Index.** Build the role, tool, store and gate tables by name. A reference to a name that is not in the document is `C_UNKNOWN_REFERENCE` (error), reported once per reference.
3. **Satisfaction.** For every requirements document in the arrangement (entry steps, delegations, verifier requirements), determine the set of roles that satisfy it, by the Capability Schema §5 predicates.
4. **Rules.** Apply §5 in the order listed, then the path properties of §6. Every finding is reported; the first does not stop the rest.
5. **Result.** `COHERENT` if no finding has severity `error`, `INCOHERENT` otherwise.

A finding is `{code, severity, subject, detail, path}`. `subject` is the name of the role, tool, store, gate or entry the finding is about, so findings are addressable. `detail` is human-readable and `path` is the chain of §6; neither is part of conformance. The code, the severity and the subject are.

---

## 8. Conformance

**Producer:** emits a document that describes its arrangement completely, and refuses at runtime everything the document says it refuses.
**Checker:** produces the specified result, and exactly the specified `{code, severity, subject}` set, for every vector in `vectors/organisation/`.

The reference checker is Rashnu's `rashnu check`, which derives a document from a live namespace or reads one from a file.

---

## 9. Relationship to the other schemas

- **Capability Schema.** A role is a declaration; an entry step and a delegation are requirements. Satisfaction is that schema's predicates, unchanged. This schema adds no matching rules of its own.
- **Approval Protocol.** A gate is an approval policy seen statically. The protocol says what must happen when an action meets one; this schema asks whether the gate could ever be satisfied at all.
- **Authority Envelope.** The envelope is authority at run time, narrowing as work passes down. `C_AUTHORITY_GAP` is the same rule checked before anything runs.
- **Decision Trace Format.** The trace records what an arrangement did. This document records what it permits. A deployment that has both can compare them: declared paths never taken, and refusals that point at a missing declaration.

---

## 10. What this cannot establish

That an arrangement is coherent does not mean its agents are competent, that its declarations are true, or that a model driving a role will behave. It means the arrangement contains no path that is impossible by construction, and no consequential action that nobody was ever asked about. What an agent chooses within a coherent arrangement is the subject of the decision trace, after the fact.
