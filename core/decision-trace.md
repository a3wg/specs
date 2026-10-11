# A3 Decision Trace Format

**Version:** 1.0.0
**Status:** Stable. Changes follow the versioning rules in the [README](../README.md#conventions).
**Licence:** Community Specification License 1.0 ([LICENSE.md](../LICENSE.md)). Independent implementation is explicitly permitted and encouraged.

A decision trace is the complete, tamper-evident record of one task carried out by an autonomous agent: what it was asked, on whose authority, what it decided, what it did, what it cost, and who approved what. This document specifies the format so that a trace can be **verified by a party that has only the trace and a set of public keys**, and so that a **different implementation can inspect and replay it** without access to the system that produced it.

The words MUST, SHOULD and MAY are used as in RFC 2119.

---

## 1. Design goals

| Goal | How the format achieves it |
|---|---|
| **Answerability** — "why did the agent do X?" is answerable from the trace alone | Every decision, policy evaluation, approval, tool call, model call and memory citation is a record |
| **Tamper evidence** — removal or alteration of any record is detectable | Records are hash-chained; each record is signed |
| **Verifiable redaction** — content can be erased after the fact without breaking verification | The chain commits to a hash of each record's content, not to the content itself |
| **Portability** — verifiable and replayable by an implementation that did not produce it | JSON, canonical hashing, documented record kinds, self-describing key set |
| **Interoperability** — does not compete with existing telemetry standards | Model-call fields reuse OpenTelemetry GenAI attribute names verbatim |

A trace proves what the emitting system *recorded*. It does not by itself prove that an external action occurred; that is the emitter's honesty plus the signing key's custody (§11).

---

## 2. Encoding and primitives

| Concern | Rule |
|---|---|
| Serialisation | JSON, UTF-8. Interchange form is **JSON Lines**: one record per line, in ascending `seq`, file extension `.jsonl`. |
| Canonicalisation | **RFC 8785 (JSON Canonicalization Scheme)** for every hash input. |
| Hash | SHA-256. Encoded as the string `sha256:` followed by 64 lowercase hex characters. The prefix allows future agility. |
| Signature | **Ed25519** over the UTF-8 bytes of the `record_hash` string. Encoded as base64url without padding. |
| Timestamps | RFC 3339, UTC (`Z` suffix), millisecond precision. |
| Identifiers | Opaque strings. `trace_id` SHOULD be a UUIDv7 so traces sort by creation time. |
| Numbers | Integers where the spec says integer. Currency as decimal strings (e.g. `"0.0421"`), never floats. |

---

## 3. Record structure

A trace is a sequence of **records**. Every record has the same envelope; the `content` object varies by `kind`.

### 3.1 Fields

| Field | Type | Hashed | Required | Meaning |
|---|---|---|---|---|
| `v` | string | yes | yes | Format version, `MAJOR.MINOR`. `"1.0"` for this document. A verifier accepts every `1.x`, and MAY accept `"0.1"`, the draft that preceded 1.0, whose records are hashed and signed identically. |
| `trace_id` | string | yes | yes | Identifies the trace. Identical in every record of the trace. |
| `seq` | integer ≥ 0 | yes | yes | Position in the chain. Starts at 0, increments by exactly 1. |
| `ts` | string | yes | yes | Time the record was created. Non-decreasing within a trace. |
| `kind` | string | yes | yes | Record kind (§4). |
| `actor` | object | yes | yes | Who did the thing recorded: `{"type": "principal" \| "agent" \| "component", "id": string}`. |
| `kid` | string | yes | yes | Identifier of the key that signed this record (§7). |
| `class` | string | yes | no | What kinds of regulated or sensitive data the content carries, as a comma-separated list of class names. Set by an emitter that classifies; how it classifies is the implementation's own. Absent when nobody classified. Opaque to the format: a verifier checks that it is covered by the record hash and never interprets it. |
| `parent` | object | yes | seq 0 of a child trace only | `{"trace_id": string, "seq": integer}` — the parent's `delegation.spawned` record (§8). MUST be absent on all other records. |
| `content_hash` | string | yes | yes | SHA-256 of the canonicalised `content` object. |
| `prev_hash` | string \| null | yes | yes | `record_hash` of the record at `seq − 1`. `null` only when `seq` is 0. |
| `record_hash` | string | — | yes | SHA-256 of the canonicalised **hashed fields** (§3.2). |
| `sig` | string | — | yes | Signature over `record_hash` by key `kid`. |
| `content` | object \| null | no | yes | The record's payload. `null` if and only if the record has been redacted (§6). |
| `redacted` | boolean | no | no | `true` if `content` has been removed under §6. Absent means `false`. |

**Why the split between hashed and unhashed fields matters:** `record_hash` covers `content_hash` but not `content`. That is what makes redaction possible — the content can be deleted while the hash it committed to stays in the chain — and it is why a verifier checks content in two separate steps (§9).

### 3.2 Computing the hashes and signature

1. `content_hash = sha256(JCS(content))`.
2. Build the **hash input**: an object containing exactly the hashed fields present on the record — `v`, `trace_id`, `seq`, `ts`, `kind`, `actor`, `kid`, `content_hash`, `prev_hash`, and `class` and `parent` if present. Nothing else.
3. `record_hash = sha256(JCS(hash_input))`.
4. `sig = Ed25519_sign(private_key[kid], utf8(record_hash))`.

An emitter MUST compute `content_hash` before any part of `content` can be modified or logged elsewhere, and MUST sign at the point of emission — for a supervisor, on the machine that runs the instance; for a gateway, in the gateway. Signing centrally after transport defeats the purpose.

### 3.3 Header identifiers must be pseudonymous

The hashed fields cannot be redacted. Therefore `actor.id` and every identifier in the envelope MUST be a stable pseudonymous identifier — the `sub` claim as issued by the identity provider, a workload identity, a component name — and MUST NOT be a name, email address or other direct identifier. Direct identifiers, where needed at all, belong in `content`, where they can be redacted.

---

## 4. Record kinds

Kinds are namespaced `<domain>.<event>`. The kinds below are defined by this version. Implementers MAY emit additional kinds prefixed `x-<vendor>.`; a verifier MUST chain-verify unknown kinds and MAY ignore their content.

Each kind lists its **required** content fields. Emitters MAY add fields. Where a field is marked *or hash*, the emitter MAY record `<field>_hash` (SHA-256 of the canonical value) instead of the value, for classified data held elsewhere; a replaying implementation then needs access to that store.

Where a field marked *or hash* is **absent entirely**, the emitter recorded neither, and the record MUST say why: `recording` carries the disposition that removed it (`omit` or `refuse`) and `recording_reason` names the rule. A verifier MUST accept the absence of a field marked *or hash* on a record carrying `recording`, and MUST NOT extend that to any other required field — a policy may remove what a tool said and may not remove that a tool was called. Without `recording`, an absent field is a defect (`E_KIND_RULE`), because "we did not write this down on purpose" and "the emitter is broken" must not be the same record. Without them, "policy removed this" and "the tool returned nothing" are the same record, and only one of them means the tool is working. An emitter that compared the content against a declaration MAY also record `undeclared_classes`, the classes present that the tool did not declare, and `classification_failed` when the classifier could not be reached — an unclassified window that leaves no evidence is indistinguishable afterwards from a window in which nothing sensitive happened.

None of these are a new encoding: a verifier that has never heard of them chain-verifies the record like any other, and an existing reader sees a trace whose results are recorded by hash, which the format has always allowed.

### 4.1 Lifecycle

| Kind | Required content | Notes |
|---|---|---|
| `trace.opened` | `format` (`"a3-trace/1.0"`), `task_id`, `tenant` | MUST be `seq` 0. MAY include `otel: {trace_id, span_id}` for correlation. |
| `task.submitted` | `goal` *or hash*, `inputs` *or hash*, `priority`, `deadline`, `idempotency_key` | |
| `task.admitted` | `originating_principal: {type, id}`, `agent: {name, namespace}`, `authority_envelope: {clearances: [string], expires}`, `budget: {tokens, usd, wall_clock_ms, actions}` | The envelope recorded here is the effective authority for the whole task. `originating_principal.type` MUST be `human` or `service`, never `agent`. |
| `task.rejected` | `reason`, `policy_ref: {seq}` | Terminal. |
| `task.bound` | `instance`, `runtime`, `scheduling: {filters_passed: [string], filters_failed: [{candidate, predicate}], scores: object}` | Explains the placement decision; a resumed placement MAY instead carry `scheduling.recorded_in`, naming the earlier `task.bound` that explained it (Capability Schema §6). MAY include `agent_revision` and, after a `task.preempted` or `task.parked`, `resumed: true`; after a `task.parked`, `parked_on` names the approval request the task waited on, or `parked_on_task` the sub-task. MAY include `instance_recycled: true` and `instance_sessions_before` (integer) when the instance had served other sessions before this task's, having emptied its writable storage between them. A trace has one `task.bound` per placement, so a resumed task has more than one. MAY include `runtime_attestation: {attestor, format, verified, measurements, statement_sha256}` when an attestor vouched for the machine the task was placed on; `verified` is that attestor's verdict and nothing more, and a reader MUST NOT take its absence as a failed attestation rather than as no attestor having been asked. |
| `context.snapshot` | `tools: [{name, version, side_effect}]`, `policies: [{name, version}]`, `model_profile`, `memory_scopes: [string]` | Emitted at bind and whenever the set changes. Required for replay (§10). |
| `task.completed` | `outcome` (`"success"` \| `"partial"`), `result` *or hash*, `usage: {tokens, usd, wall_clock_ms, actions}` | Terminal. |
| `task.failed` | `reason`, `failure_class` | Terminal. `failure_class` SHOULD be one of `guardrail`, `budget`, `policy`, `tool`, `model`, `internal`. |
| `task.cancelled` | `by: {type, id}`, `reason` | Terminal. |
| `trace.closed` | `record_count`, `usage` | MUST be the last record other than `redaction.applied` (§5). |

### 4.2 Policy and approval

| Kind | Required content | Notes |
|---|---|---|
| `policy.evaluated` | `point` (`admission` \| `pre-action` \| `post-action`), `policy`, `rule`, `inputs_hash`, `decision` (`allow` \| `deny` \| `require-approval`), `reason` | One record per evaluation, including allows. |
| `plan.proposed` | `plan_hash`, `action: {tool, operation, arguments}` *or hash*, `side_effect`, `reasoning_summary` | `plan_hash` is computed as the Approval Protocol specifies; until that specification exists, `sha256(JCS(action))`. |
| `approval.requested` | `request_id`, `plan_hash`, `policy`, `approvers_from: [string]`, `quorum`, `expires` | MAY include `dry_run` (hash of the attached dry-run result), `escalates` (the `request_id` this one replaces) and `escalation: {to_groups, timeout_ms}`. |
| `approval.decided` | `request_id`, `plan_hash`, `decision` (`approve` \| `deny` \| `expired` \| `break-glass`), `approver: {id}`, `step_up` (boolean) | `plan_hash` MUST equal the one in the matching request. |
| `approval.notified` | `request_id`, `event` (`raised` \| `escalated` \| `reminder` \| `decided` \| `expired`), `approvers_from`, `delivered` (boolean), `notification_id` | Who was told a decision was waiting, and whether telling them worked. A failed delivery MUST be recorded, not dropped: it is the difference between a gate nobody heard about and a gate nobody cared about. It MUST NOT carry anything the notification itself may not carry (`spec/interfaces.md` §11). |

### 4.3 Actions

| Kind | Required content | Notes |
|---|---|---|
| `tool.invoked` | `call_id`, `tool`, `operation`, `arguments` *or hash*, `side_effect`, `plan_hash` (if approval-gated) | MUST NOT contain a credential. MAY contain `credential_lease_id`, `clearances_required`, and `dry_run: true` when the invocation was a dry run performed for an approval (then `side_effect` is `read`). |
| `tool.completed` | `call_id`, `status` (`ok` \| `error` \| `refused`), `result` *or hash*, `duration_ms`, `trust_level` | `trust_level` is the label the result carried into agent context, e.g. `tool-result`. |
| `model.inferred` | `gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `prompt` *or hash*, `response` *or hash*, `fallback` (boolean) | Field names are OpenTelemetry GenAI semantic-convention attributes, used verbatim (§12). `gen_ai.usage.input_tokens` is every token the provider read, cached or not. MAY include `cost_usd`, `provider` and `residency` when a model gateway bound the call; with `cost_usd` an emitter SHOULD include `cost_exact` (boolean), which is `false` when the price used did not cover everything the provider reported — see §4.4. MAY include `gen_ai.usage.cache_read_tokens` and `gen_ai.usage.cache_write_tokens`, which are the part of the input count served from, or written to, a prompt cache. An inference served incrementally MAY carry `stream: true` with `ttft_ms` and `stream_chunks`; where it does, `truncated: true` with `truncated_reason` says the agent was handed part of an answer rather than all of one, and `usage_reported: false` says the provider never said what it charged for, which is not the same as a call that cost nothing. `prompt`/`response` and their hashes are over the whole prompt and the assembled answer either way, so the same answer hashes the same whether it arrived at once or in pieces. |
| `memory.retrieved` | `records: [{store, id}]`, `query_hash` | Cites memory by identifier so a decision can be attributed to the context that informed it. |
| `memory.written` | `store`, `id`, `scope` | |

### 4.4 Limits and delegation

| Kind | Required content | Notes |
|---|---|---|
| `budget.consumed` | `dimension` (`tokens` \| `usd` \| `wall_clock_ms` \| `actions`), `amount`, `remaining` | Emitters MAY aggregate; SHOULD emit at least once per model or tool call. |
| `budget.exceeded` | `dimension`, `limit`, `action` (`suspend` \| `degrade` \| `fail`) | |
| `guardrail.tripped` | `rule` (`loop` \| `depth` \| `fanout` \| `tool_calls` \| `egress`), `detail` | |
| `delegation.spawned` | `child_trace_id`, `child_task_id`, `agent: {name, namespace}`, `authority_envelope` | The child's envelope MUST be a subset of the parent's (§8). |
| `delegation.completed` | `child_trace_id`, `outcome` | |
| `task.checkpointed` | `state` *or hash*, `progress` | Emitted by the agent when asked to stop before finishing (preemption or drain). Not terminal. |
| `task.preempted` | `reason`, `checkpoint_seq` (integer or null) | Emitted by the supervisor after the instance stopped. Not terminal: the task resumes with a new `task.bound`, in the same trace, from the checkpoint if one was recorded. |
| `task.rebound` | `reason`, `rebinds` (integer) | Emitted by the scheduler when the instance a task was bound to disappeared before the task finished and no terminal record was written. Not terminal: the task returns to the queue and resumes with a new `task.bound`, in the same trace. Unlike `task.preempted` the instance was not asked to stop, so there is no checkpoint and the turn restarts from what the memory service holds. A task that has already emitted a consequential `tool.invoked` (§4.4) MUST NOT be rebound; it fails instead, and the reason MUST say so. Implementations MUST bound the number of rebinds. |
| `task.parked` | `approval_request` or `sub_task`; `held_ms` (integer) | Emitted by the supervisor when a tool call has waited for an approval (`approval_request`), or a delegation for its sub-task (`sub_task`), for the configured hold and the task gives its instance back. Not terminal: the task resumes with a new `task.bound`, in the same trace, once the request — or the one it escalated to — is decided, or the sub-task has ended. A resumed delegation MUST NOT spawn its sub-task again. The resumed instance MUST NOT repeat an action taken before the park and MUST NOT write again a record already in the trace; it fails the task rather than proceed if it cannot establish that the agent is repeating exactly what it did before. What it resumes from is not part of the trace (§4.6). |

### 4.5 Redaction

| Kind | Required content | Notes |
|---|---|---|
| `redaction.applied` | `targets: [integer]` (seqs), `reason`, `authority: {type, id}` | The only record kind permitted after `trace.closed`. MUST NOT target `seq` 0, a `trace.closed` record, or another `redaction.applied` record. |

---

### 4.6 Replay records are not part of the trace

A parked task (`task.parked`) resumes from a **replay record**: every request its agent made and the answer it was given before the park. It is not a trace record and appears in no trace. It is kept apart because it has the opposite properties: it must hold exact bytes where a trace holds what the recording policy allows, and it is needed for minutes where a trace is kept for years.

An implementation that keeps replay records:

- MUST encrypt them before storing them, under a key the trace store does not hold, and MUST bind each to the namespace and trace it belongs to so that it cannot be opened for another.
- MUST make a record readable only by the instance the task is currently bound to.
- MUST delete a record when its task ends, and SHOULD delete it with its trace.
- MUST NOT store one unencrypted. Without a key a task cannot be parked; an implementation SHOULD refuse to start rather than fall back to holding instances indefinitely.

A verifier does not read replay records. Everything a resumed task did is in the trace, and verifies as any other.

## 5. Chain rules

1. `seq` 0 MUST be `trace.opened` with `prev_hash: null`. No other record may have `prev_hash: null`.
2. `seq` MUST increase by exactly 1; `prev_hash` MUST equal the previous record's `record_hash`.
3. `ts` MUST be non-decreasing.
4. `trace_id` MUST be identical across all records.
5. After `trace.closed`, only `redaction.applied` records MAY follow.
6. A trace with no `trace.closed` is **open**, not invalid. A verifier reports it as such.
7. A record whose `kind` is terminal (`task.completed`, `task.failed`, `task.cancelled`, `task.rejected`) MUST be followed only by `trace.closed` and `redaction.applied`.

### 5.1 Emission and ingest are different times

`ts` is when a record was sealed, not when it reached a store, and this format does not record the second at all. A trace MAY therefore be written by an emitter that is disconnected — a device in the field, a machine on a private network, a vehicle between one depot and the next — held for as long as that lasts, and shipped afterwards. It verifies identically, because every check in §9 reads the records and the key set and none of them reads the clock.

A verifier MUST NOT reject a record because `ts` is old, and MUST NOT compare `ts` against the time of verification or of ingest. Step 6 evaluates key validity against `ts` for exactly this reason, and that is deliberate rather than incidental: a key that expired while an emitter was offline still verifies the records it signed while it was valid.

Two things change for a deployment whose emitters may be disconnected, and both are consequences of rules stated elsewhere rather than new ones.

- **Revocation stops reaching them.** §7.2 says revocation bounds a key's honest use rather than proving when a record was made. A disconnected emitter is where that gap is widest, because nothing can be withdrawn from it between one connection and the next. Validity windows SHOULD be short and enrolment SHOULD be per emitter, since the window is the only bound that still applies.
- **The completeness control weakens.** §11 closes truncation from outside the file, and the first thing it offers is the store's own head, carrying the record count. That is an independent witness of a trace's length only for records that have arrived; about a trace still held on an emitter it witnesses nothing, and a partial upload is indistinguishable from a truncation. A deployment that needs completeness from such an emitter MUST have the emitter commit to its own length — a periodic signed checkpoint carrying the record count is the smallest thing that does it — because the store cannot do it on the emitter's behalf.

A verifier cannot tell an open trace from a partially uploaded one, and MUST NOT try: both are `VALID_OPEN`, and which one it is, is a fact about the emitter that the records do not carry.

---

## 6. Verifiable redaction

Redaction removes a record's **content** while leaving its place in the chain intact and provable.

**To redact record N:**

1. Set `content` to `null` and `redacted` to `true` on record N. Leave every hashed field, `record_hash` and `sig` unchanged.
2. Append a `redaction.applied` record whose `targets` includes N, stating the reason and the authority under which it was done. This record is chained and signed like any other.
3. Erase the content from every store and index that held it.

**What the verifier can then establish:** that record N existed, when, of what kind, by which actor, and that its content hashed to `content_hash` — without knowing the content. And that the redaction itself was recorded, by whom, and why.

**Rules:**

- A redacted record MUST have a corresponding `redaction.applied` record later in the chain. A redacted record without one is a verification failure (`E_UNRECORDED_REDACTION`): silent deletion is exactly what this construction exists to detect.
- Redaction MUST NOT rewrite, remove or re-chain any record. History is only ever appended to.
- Records at `seq` 0, `trace.closed` records and `redaction.applied` records are not redactable.
- Deletion and erasure obligations that fall on a trace MUST be met this way and no other.
- An implementation MUST check that the `redaction.applied` record satisfies §3 **before** step 1. A chain is append-only, so a malformed record appended here makes the trace report `INVALID` from then on, permanently, with the content already erased — every later verification then reports tampering where there was none. This is the one append that has to be checked before it is made rather than after, and it is the obvious place to get it wrong: the authority that *ordered* an erasure is often a person, and `actor.type` has no such value, because a person does not erase a record — they order it erased, and something holding a signing key carries it out. The orderer belongs in the content, and the actor is what performed it.

---

## 7. Keys and signing

### 7.1 Key set

A verifier is given a **key set** document:

```json
{
  "keys": [
    {
      "kid": "supervisor-node-a1-2026-09",
      "alg": "Ed25519",
      "pub": "<base64url public key>",
      "component": "supervisor",
      "valid_from": "2026-09-01T00:00:00.000Z",
      "valid_to": "2026-12-01T00:00:00.000Z",
      "revoked_at": "2026-10-04T09:12:00.000Z",
      "revocation_reason": "node a1 seized; key presumed copied"
    }
  ]
}
```

`revoked_at` is optional and absent in the ordinary case.

### 7.2 Rules

- Every record MUST be signed. Each emitting component (supervisor, gateway, control plane) signs with its own key; the `kid` names it.
- A record's `ts` MUST fall within the signing key's validity window.
- Keys SHOULD rotate; a rotated key stays in the key set for as long as traces signed by it must remain verifiable.
- A key MAY carry `revoked_at`. A verifier MUST reject a record whose `ts` is at or after it (`E_KEY_REVOKED`), and MUST still accept records signed before it. Revocation and expiry are different facts and a verifier MUST report them differently: a key used outside its window is usually a clock or a rotation problem, while a revoked key means somebody decided the key was no longer trustworthy, and the two lead a reader to look in different places.
- Revoking a key MUST NOT remove it from the key set. The records it signed before the revocation still have to verify, and a verifier that cannot find the key reports `E_KEY_UNKNOWN`, which says nothing about what happened.
- **What revocation does and does not bound.** `ts` is chosen by whatever wrote the record, so a holder of a stolen key can claim a time before the revocation. Revocation bounds a key's *honest* use and makes dishonest use visible as a claim rather than as a fact; it is not by itself proof of when a record was made. A deployment that needs that ordering takes it from something the signer does not control — the sink's own append order, a timestamping authority, or a transparency log — and that anchor is outside this format.
- Private keys MUST be held by the emitting component and MUST NOT be exportable through any interface of the implementation.
- The key set is distributed out of band. How it is itself trusted (a signed key set, a transparency log, an external KMS) is a deployment decision, not part of this format. Implementations SHOULD support at least a self-contained key set so that verification has no external prerequisite.

---

## 8. Delegation trees

When a task delegates to another agent, the child runs under its own trace.

- The parent emits `delegation.spawned` with `child_trace_id`.
- The child's `seq` 0 carries `parent: {trace_id, seq}` pointing at that record.
- Given a **bundle** of traces, a verifier MUST check both directions of the link and MUST check that the child's `authority_envelope` in its `task.admitted` record is a subset of the parent's: every clearance in the child appears in the parent, and every budget dimension in the child is ≤ the parent's remaining amount at the point of delegation. A child holding more than its parent is `E_ENVELOPE_WIDENED`.

---

## 9. Verification procedure

Input: one trace (`.jsonl`), optionally a bundle of related traces, and a key set.

For each record in order:

1. **Structure.** Required fields present, types correct, `v` supported. Else `E_STRUCTURE`.
2. **Sequence.** `seq` is exactly previous + 1 (or 0 for the first). Else `E_SEQ`. After a mismatch the verifier resynchronises on the record's own `seq`, so a single removed record produces one `E_SEQ` at the gap rather than one for every record after it (vector 05).
3. **Genesis.** If `seq` 0: kind is `trace.opened` and `prev_hash` is `null`. Otherwise `prev_hash` is not `null`. Else `E_GENESIS`.
4. **Chain.** `prev_hash` equals the previous record's `record_hash`. Else `E_PREV_HASH`.
5. **Record hash.** Recompute per §3.2 and compare. Else `E_RECORD_HASH`.
6. **Key.** `kid` is in the key set (`E_KEY_UNKNOWN`), `ts` is within its validity (`E_KEY_EXPIRED`), and `ts` is before any `revoked_at` (`E_KEY_REVOKED`).
7. **Signature.** Verify `sig` over `record_hash` with the key. Else `E_SIG`.
8. **Content.** If not redacted: `content` is non-null and `sha256(JCS(content))` equals `content_hash`. Else `E_CONTENT_HASH`. If redacted: `content` is `null`; remember `seq` for step 12.
9. **Time.** `ts` ≥ previous `ts`. Else `E_TS_ORDER`.
10. **Kind rules.** Kind-specific content requirements (§4) hold; no record other than `redaction.applied` follows `trace.closed`; nothing but `trace.closed` or `redaction.applied` follows a terminal kind. Else `E_KIND_RULE` or `E_AFTER_CLOSE`.

After the last record:

11. **Closure.** If no `trace.closed`: result is `VALID_OPEN` rather than `VALID`, provided all other checks pass.
12. **Redactions.** Every redacted record's `seq` appears in the `targets` of some later `redaction.applied` record. Else `E_UNRECORDED_REDACTION`. No `redaction.applied` targets a non-redactable record. Else `E_REDACTION_TARGET`.
13. **Delegation** (bundle only). Links are bidirectional and envelopes are subsets (§8). Else `E_PARENT_LINK` / `E_ENVELOPE_WIDENED`.

**Result:** `VALID`, `VALID_OPEN`, `EMPTY`, or `INVALID` with the list of every error found, each naming the `seq`. A verifier MUST report all errors, not stop at the first.

**Nothing is a fourth answer.** A trace begins with `trace.opened` at `seq` 0, so an input holding no records is not a trace, and a verifier MUST report `EMPTY` for it rather than `VALID_OPEN`. Otherwise a check pointed at a file that was never written — a failed fetch, a path that does not exist, a store that was not the one holding the records — answers "valid" and exits zero, which is the one answer a verification gate must never give by accident. `EMPTY` is not `INVALID` either: a sink creates a trace before anything is appended to it, so a store holds genuinely empty traces in the ordinary course of a task starting, and treating those as tampered would be a false alarm every time. The distinction is the caller's to act on: a command line verifying a file treats `EMPTY` as a failed read, and a sweep over a store treats it as nothing to check yet.

In a bundle the worst result wins, and `EMPTY` outranks `VALID_OPEN`: a bundle is a claim to hold a whole chain of authority, and a member holding nothing means no claim is being made about that trace at all.

---

## 10. Replay sufficiency

A conformant trace carries enough for a different implementation to reconstruct, at any `seq`, what the agent could see and do. Concretely, at every point the following are recoverable from records at or before that `seq`:

- the task's goal and inputs (`task.submitted`),
- the effective authority and budget (`task.admitted`, `budget.consumed`),
- the tools, policies, model profile and memory scopes available (`context.snapshot`),
- every model prompt and response, or their hashes with a reference to where the content is held (`model.inferred`),
- every memory record consulted, by identifier (`memory.retrieved`),
- every policy decision and its rule (`policy.evaluated`),
- every proposed action, its approval status and outcome (`plan.proposed`, `approval.*`, `tool.*`).

Replay is for *inspection*: reconstructing the situation, not reproducing identical model output.

---

## 11. Security considerations

- **What is proven.** A valid trace proves that the holder of key `kid` recorded these records in this order, and that no record has been altered, removed from between two others, or reordered since. It does not prove the emitter told the truth, and it does not prove the trace is *complete* — see the next bullet. The trust boundary is the emitting component and its key.
- **Truncation is not detectable from the file alone.** A chain commits each record to the one before it, so nothing can be removed from the middle. Nothing commits to how long the chain should be, so records can be removed from the **end**. Delete `trace.closed` and the records before it and every remaining record still hashes, chains and verifies; by §5 rule 6 the result is an open trace, which is legitimate, and a verifier reports `VALID_OPEN` with no errors. An approval denial, a `guardrail.tripped` or a `task.failed` can be taken off the end of a closed trace and the file will not say so. This is a property of any hash chain that does not commit to its own length, and it cannot be closed inside the format: a verifier is handed the file and has nothing to compare it against. It is closed from outside, by something that recorded the length independently — the store's own head, which carries the record count and whether the trace was closed; a copy exported beforehand and compared against what the store now holds (Rashnu does this with `rashnu traces audit`); or the external anchor §7.2 describes and defers. A deployment that needs completeness as well as integrity MUST use one of them.
- **Sign at the source.** Records MUST be signed where they are created, so that what arrives at storage is already committed. Redaction of secrets (below) MUST happen before hashing.
- **Secrets.** Credentials MUST never appear in any record. Emitters MUST redact known secret material before computing `content_hash`; a secret that was hashed into a record can only be removed by redacting the whole record.
- **Header exposure.** Hashed fields are permanent. See §3.3.
- **Key compromise.** A compromised key allows forged records for its validity window only if the attacker can also append to the chain in storage; append-only storage with independent access control is the complementary control. Key rotation limits the window.
- **Timestamps.** `ts` is emitter-asserted. Where independent time is required, an external timestamping extension (e.g. RFC 3161 over a checkpoint's `record_hash`) MAY be layered on; it is out of scope for 0.1.
- **Disconnected emitters.** Where records are sealed long before they are stored, both of the controls above lose reach: revocation cannot be delivered, and the store's head cannot witness the length of a trace it has not received. See §5.1.

---

## 12. Relationship to OpenTelemetry

This format is a **record of authority and decision**; OpenTelemetry is a **record of execution and performance**. They are complementary and should be correlated, not merged.

- `model.inferred` uses OpenTelemetry GenAI semantic-convention attribute names verbatim (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, …). This format MUST NOT redefine the meaning of any `gen_ai.*` attribute.
- `trace.opened` MAY carry `otel: {trace_id, span_id}`, and any record MAY carry the same in `content`, so that a decision trace and an OpenTelemetry trace can be joined.
- Nothing in this format is intended to be emitted *as* OpenTelemetry spans. Where a field is covered by an existing convention, this format defers to it.

---

## 13. Conformance

Two conformance roles:

- **Emitter:** produces traces that a conformant verifier reports as `VALID` or `VALID_OPEN`, including correct redaction behaviour.
- **Verifier:** produces the specified result and error codes for every vector in `vectors/decision-trace/`.

The test vectors are normative. Each is a `.jsonl` trace, a key set, and the expected result. The initial set covers: a valid closed trace; a valid open trace; one byte altered in content; one byte altered in a hashed field; a removed record; a reordered record; an unknown key; a signature from the wrong key; a key used outside its validity; a record signed after its key was revoked, and one signed before it by the same key; a correctly redacted record; a redacted record with no `redaction.applied`; a redaction targeting `seq` 0; a record after `trace.closed`; a delegation bundle with a widened envelope.

---

## 14. Example

A minimal closed trace: a read-only task, one tool call, one model call. Hashes and signatures are placeholders.

```json
{"v":"1.0","trace_id":"019267b4-2a1e-7c3d-8e5f-6a7b8c9d0e1f","seq":0,"ts":"2026-09-16T14:00:00.000Z","kind":"trace.opened","actor":{"type":"component","id":"control-plane"},"kid":"cp-2026-09","content_hash":"sha256:…","prev_hash":null,"record_hash":"sha256:…","sig":"…","content":{"format":"a3-trace/1.0","task_id":"task-4471","tenant":"support"}}
{"v":"1.0","trace_id":"019267b4-…","seq":1,"ts":"2026-09-16T14:00:00.012Z","kind":"task.submitted","actor":{"type":"principal","id":"sub|8f3a…"},"kid":"cp-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"goal":"Summarise what happened to order A-1043","inputs":{"order":"A-1043"},"priority":"normal","deadline":"2026-09-16T14:10:00.000Z","idempotency_key":"9c1e…"}}
{"v":"1.0","trace_id":"019267b4-…","seq":2,"ts":"2026-09-16T14:00:00.020Z","kind":"task.admitted","actor":{"type":"component","id":"policy"},"kid":"cp-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"originating_principal":{"type":"human","id":"sub|8f3a…"},"agent":{"name":"order-analyst","namespace":"support"},"authority_envelope":{"clearances":["orders.read","customers.read"],"expires":"2026-09-16T14:10:00.000Z"},"budget":{"tokens":400000,"usd":"0.75","wall_clock_ms":600000,"actions":40}}}
{"v":"1.0","trace_id":"019267b4-…","seq":3,"ts":"2026-09-16T14:00:00.150Z","kind":"task.bound","actor":{"type":"component","id":"scheduler"},"kid":"cp-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"instance":"order-analyst-7d9f","runtime":"node-a1","scheduling":{"filters_passed":["capability:order.investigate/2","clearance:orders.read","residency:eu"],"filters_failed":[],"scores":{"memory_affinity":0.0,"cost":0.8}}}}
{"v":"1.0","trace_id":"019267b4-…","seq":4,"ts":"2026-09-16T14:00:00.160Z","kind":"context.snapshot","actor":{"type":"component","id":"supervisor"},"kid":"supervisor-node-a1-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"tools":[{"name":"order-lookup","version":"1.2.0","side_effect":"read"}],"policies":[{"name":"support-default","version":"14"}],"model_profile":"interactive-eu","memory_scopes":["session"]}}
{"v":"1.0","trace_id":"019267b4-…","seq":5,"ts":"2026-09-16T14:00:01.300Z","kind":"tool.invoked","actor":{"type":"agent","id":"order-analyst-7d9f"},"kid":"gateway-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"call_id":"c-01","tool":"order-lookup","operation":"order_status","arguments":{"order":"A-1043"},"side_effect":"read"}}
{"v":"1.0","trace_id":"019267b4-…","seq":6,"ts":"2026-09-16T14:00:01.480Z","kind":"tool.completed","actor":{"type":"component","id":"gateway"},"kid":"gateway-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"call_id":"c-01","status":"ok","result_hash":"sha256:…","duration_ms":180,"trust_level":"tool-result"}}
{"v":"1.0","trace_id":"019267b4-…","seq":7,"ts":"2026-09-16T14:00:03.900Z","kind":"model.inferred","actor":{"type":"agent","id":"order-analyst-7d9f"},"kid":"supervisor-node-a1-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"gen_ai.system":"self-hosted","gen_ai.request.model":"summariser-8b","gen_ai.response.model":"summariser-8b","gen_ai.usage.input_tokens":1840,"gen_ai.usage.output_tokens":212,"prompt_hash":"sha256:…","response_hash":"sha256:…","fallback":false,"cost_usd":"0.0000"}}
{"v":"1.0","trace_id":"019267b4-…","seq":8,"ts":"2026-09-16T14:00:03.950Z","kind":"task.completed","actor":{"type":"component","id":"supervisor"},"kid":"supervisor-node-a1-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"outcome":"success","result_hash":"sha256:…","usage":{"tokens":2052,"usd":"0.0000","wall_clock_ms":3950,"actions":1}}}
{"v":"1.0","trace_id":"019267b4-…","seq":9,"ts":"2026-09-16T14:00:03.960Z","kind":"trace.closed","actor":{"type":"component","id":"control-plane"},"kid":"cp-2026-09","content_hash":"sha256:…","prev_hash":"sha256:…","record_hash":"sha256:…","sig":"…","content":{"record_count":10,"usage":{"tokens":2052,"usd":"0.0000","wall_clock_ms":3950,"actions":1}}}
```

---

## 15. Settled in 1.0, and future work

Settled:

1. **`class` is hashed.** A classification cannot be quietly downgraded. A wrong label is corrected by redacting the record and recording it again.
2. **One signature per record.** A checkpoint scheme (signing every Nth `record_hash` and the final one) would be cheaper. It belongs to the planned compact profile, which can add it without changing this format.
3. **`budget.consumed` is SHOULD per model and tool call.** An emitter MAY aggregate. A verifier that reconstructs remaining budget (Authority Envelope §5) works from whatever was recorded, and an emitter that under-records is answerable for it.
4. **`trust_level` is an opaque label.** It records what the emitter called the result when it entered the agent's context. A verifier checks that it is present and never interprets it. A shared vocabulary can be added in a minor version without making existing labels invalid.
