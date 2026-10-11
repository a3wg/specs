# A3 Key Discovery

**Version:** 0.1 (draft)
**Status:** Draft. No vectors yet, and nothing implements it. Everything here can change.
**Licence:** Community Specification License 1.0 ([LICENSE.md](../LICENSE.md)). Independent implementation is explicitly permitted and encouraged.
**Depends on:** [Decision Trace Format](decision-trace.md) 1.0 §7 (the key set and its rules).

Every A3 specification ends in a signature check against a **key set**: the trace's records, an authority envelope's issuer, an approval's decision. The Decision Trace Format defines what a key set contains and how it changes (§7), and then says it is "distributed out of band". That was enough while the only verifier sat next to the emitter. It is not enough once a party outside the installation has to check something:

- an **A2A partner** receiving a child envelope from another organisation's agent, before it acts on it;
- a **resource server** receiving a request with an envelope or a plan hash attached, deciding whether the approval it claims is real;
- an **offline verifier**, an auditor a year later, holding a trace and needing the keys that signed it.

This specification gives all three one answer: a single document shape, at a single well-known location, with rules about how it may change. It adds nothing to the key set itself. A key-set document is a Decision Trace key set with a few members around it, and any existing verifier can read it unchanged.

The words MUST, SHOULD and MAY are used as in RFC 2119.

---

## 1. Properties

| Property | Meaning |
|---|---|
| **One location** | Given an installation's identifier, there is exactly one URL to fetch its keys from, computed without asking anyone. |
| **Self-identifying** | The document names the installation it belongs to, so it cannot be served as somebody else's. |
| **Compatible** | Its `keys` member is a Decision Trace §7.1 key set, byte for byte. A verifier that already reads key sets needs no change. |
| **Append-only** | A published key is never removed and never altered, except to record its revocation. A consumer can tell when that rule was broken. |
| **Snapshot-able** | The fetched document, saved unchanged, is a complete input for offline verification. |
| **Honest about trust** | It says exactly what fetching it establishes (§8) and no more. |

---

## 2. Terms

**Installation.** One deployment that signs records under these specifications with keys it controls: a control plane and the components that sign alongside it. One organisation may run several.

**Installation identifier.** An `https` URL that names an installation: scheme, host, optional port, optional path; no query and no fragment. For example `https://agents.example.com` or `https://example.com/tenants/acme`. It is a name first and a location second. Two installations MUST NOT share one.

**Key-set document.** The JSON document defined in §3.

**Publisher.** Whatever serves an installation's key-set document.

**Consumer.** Anything that fetches a key-set document to check a signature.

---

## 3. The key-set document

```json
{
  "format": "a3-keys/0.1",
  "installation": "https://agents.example.com",
  "updated_at": "2026-10-04T09:12:00.000Z",
  "keys": [
    {
      "kid": "cp-2026-09",
      "alg": "Ed25519",
      "pub": "w4GEKbpRth00glxRU2OwW7LU9l6ZZJkv937G5LZyc3M",
      "component": "control-plane",
      "valid_from": "2026-09-01T00:00:00.000Z",
      "valid_to": "2026-12-01T00:00:00.000Z"
    },
    {
      "kid": "supervisor-node-a1-2026-09",
      "alg": "Ed25519",
      "pub": "9Z6o55aowQVUlJ6VbyiicXy2BcqE2Reyxd6LDMTSWuE",
      "component": "supervisor",
      "valid_from": "2026-09-01T00:00:00.000Z",
      "valid_to": "2026-12-01T00:00:00.000Z",
      "revoked_at": "2026-10-04T09:12:00.000Z",
      "revocation_reason": "node a1 seized; key presumed copied"
    }
  ]
}
```

| Member | Required | Meaning |
|---|---|---|
| `format` | yes | `"a3-keys/0.1"`. A consumer MUST reject a document whose major version it does not know. |
| `installation` | yes | The installation identifier (§2). |
| `keys` | yes | A key set exactly as the Decision Trace Format §7.1 defines it, under the rules of §7.2. This specification adds no members to a key. |
| `updated_at` | no | When the publisher last changed `keys`. Informational: a consumer MUST NOT use it to decide whether a key is valid. |
| `jwks_uri` | no | Where the JWKS view (§7) is served, if the installation publishes one. |

Unknown top-level members MUST be ignored, so that later versions and bindings can add to the document. The media type is `application/a3-keys+json`; a publisher MAY serve `application/json` instead.

This specification does not change what `component` means or which components may sign what. Those rules belong to the specification that does the checking: for example, the Authority Envelope requires its `issuer` to resolve to a control-plane key.

---

## 4. Location

The key-set document of installation `https://{host}[:{port}]{path}` is served at:

```
https://{host}[:{port}]/.well-known/a3-keys{path}
```

The well-known segment goes between the authority and the path, as OAuth authorisation-server metadata does (RFC 8414 §3). It is not appended after the path. Examples:

| Installation identifier | Key-set document |
|---|---|
| `https://agents.example.com` | `https://agents.example.com/.well-known/a3-keys` |
| `https://example.com/tenants/acme` | `https://example.com/.well-known/a3-keys/tenants/acme` |

A consumer fetches it with `GET` over HTTPS and:

1. MUST reject anything but a `200` response whose body parses as §3. (`D_FETCH`, `D_FORMAT`)
2. MUST reject the document unless its `installation` equals, character for character, the identifier the consumer used to build the URL. (`D_INSTALLATION_MISMATCH`) This check is what stops one installation from serving its keys as another's, and it is not optional.
3. MAY follow redirects only to the same scheme, host and port.

The well-known name `a3-keys` will be registered under RFC 8615 when this specification reaches draft. Until then it is provisional.

### 4.1 Referring to a key set from elsewhere

Other documents point at an installation's keys in one of two ways:

- by its **installation identifier**, from which the URL follows by §4; or
- by a member named **`a3_keys_uri`** holding the full URL of the key-set document, for formats that hold URLs rather than names (OAuth authorisation-server or protected-resource metadata, an A2A Agent Card extension).

Bindings say where these pointers go in their own protocol. This specification fixes only the member name and what it means. Following a pointer does not remove the check in §4 step 2: the document still has to name the installation the consumer meant.

---

## 5. Change over time

A trace is verified long after it is written, against whatever the publisher serves at the time. The rules in Decision Trace Format §7.2 that make this safe are restated here as obligations on the publisher, because a document served on the network is where they get broken.

A publisher:

- MUST keep every key in the document for as long as anything it signed must remain verifiable. Rotation adds a key under a new `kid`; it never replaces one.
- MUST NOT change any member of a published key except to add `revoked_at` and `revocation_reason`. Correcting a mistake in a key means revoking it and publishing a new one.
- MUST NOT reuse a `kid`, even for a key that was never used to sign.

A consumer that caches a document:

- SHOULD retain every key it has seen for an installation.
- MUST NOT quietly adopt a newer document that drops a key it has seen (`D_KEY_REMOVED`) or changes a published key other than by adding a revocation (`D_KEY_CHANGED`). It reports the finding, and keeps verifying against what it retained. Either one means the installation broke its own record, and somebody needs to know which keys it was.
- SHOULD honour HTTP caching headers. When it meets a `kid` it does not know, it MAY refetch at once, but SHOULD do so at most once per five minutes per installation, so that a stream of records with unknown key identifiers cannot drive a stream of fetches.

---

## 6. Snapshots and offline verification

A consumer that may need to verify later SHOULD save the key-set document it fetched, unchanged, alongside what it verified, together with the time it fetched it. The saved document is a complete key set for the Decision Trace Format's verification procedure (§9 there) and needs no network.

A snapshot records what the installation served at that moment and nothing else. A later snapshot that disagrees with an earlier one under §5 is the same finding as `D_KEY_REMOVED` or `D_KEY_CHANGED`, and a verifier holding both SHOULD report it.

---

## 7. The JWKS view

OAuth resource servers and most JOSE libraries already read JSON Web Key Sets (RFC 7517). An installation MAY publish one alongside the key-set document and name it in `jwks_uri`. Each Ed25519 key appears as an Octet Key Pair (RFC 8037):

```json
{"kty": "OKP", "crv": "Ed25519", "x": "<pub, unchanged>", "kid": "<kid>", "alg": "EdDSA", "use": "sig"}
```

`pub` and `x` are both the base64url encoding, without padding, of the raw 32-byte public key, so the value carries over unchanged.

A JWKS has no validity window and no revocation. The JWKS view therefore MUST contain only keys that are valid now and not revoked, and it is for checking **fresh** signatures only: an envelope or a plan presented with a request in progress. A verifier of a trace MUST use the key-set document, never the JWKS view. Otherwise a revoked key looks the same as a key that was never published.

---

## 8. What fetching it establishes

Fetching the document over HTTPS establishes one thing: whoever controls the installation identifier's host, and a certificate for it, serves these keys now. It does not establish:

- that the keys served now are the keys served last month. A publisher can rewrite its history, and §5 only lets a consumer that cached the earlier version notice;
- who the installation belongs to beyond the domain name;
- that the consumer should trust the installation at all.

The first is the work of the trace-anchoring profile: signed key-set documents and a transparency log of their history. That profile is planned and is deliberately not part of this document. The third is the consumer's decision, made out of band: a partner agreement, configuration, a procurement process. A consumer SHOULD take the installation identifier it trusts from its own configuration, never from the data it is verifying. A trace that says "my keys are over there" is a claim, not a reason to believe it.

---

## 9. Results

A consumer reports one of:

| Code | Meaning |
|---|---|
| `D_OK` | The document was fetched, parsed, named the right installation, and is consistent with everything previously seen. |
| `D_FETCH` | Not fetched: network failure, non-`200` status, or a redirect off the origin. |
| `D_FORMAT` | Fetched but not a §3 document, or an unknown major `format`. |
| `D_INSTALLATION_MISMATCH` | `installation` is not the identifier the consumer asked for. |
| `D_KEY_REMOVED` | A key seen before is missing. |
| `D_KEY_CHANGED` | A key seen before has changed other than by revocation. |

`D_KEY_REMOVED` and `D_KEY_CHANGED` can be reported together, and alongside `D_OK` for the rest of the document. Every finding is reported, never only the first.

---

## 10. Conformance

Two conformance roles:

- **Publisher:** serves a document that satisfies §3 and §4, and never breaks §5.
- **Consumer:** implements §4's checks and §5's comparison, and produces the specified result for every vector.

There are no vectors yet. They are required before this specification can reach 1.0. They will be sequences of documents as a consumer would fetch them, each with its expected results, covering at least: a valid document; an installation mismatch; a path-bearing identifier; an unknown major format; a key removed between fetches; a key's `pub` changed; a key revoked between fetches (`D_OK`); and a JWKS view that wrongly includes a revoked key.

---

## 11. Open questions for 0.1

- **Signed key-set documents.** Should a key set be signable by a long-lived installation key, so that a snapshot proves where it came from without TLS? The current answer is that this belongs to the anchoring profile. It may turn out that every consumer wants it.
- **Algorithms.** The Decision Trace Format names only Ed25519. Constrained devices (the compact profile) may need others. The JWKS view's mapping is written for Ed25519 alone.
- **Installation identifier in the trace.** A trace does not record which installation emitted it. Adding it would make a stray trace self-locating, but it is a change to the core, and §8 says why it would not be a trust anchor anyway. For now, bindings carry it where they need it.
