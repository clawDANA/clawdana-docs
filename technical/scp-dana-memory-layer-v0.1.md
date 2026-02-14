# SCP/DANA Memory Layer — mini‑RFC v0.1 (sketch)

**Status:** draft / implementable sketch

## 0. Purpose

Provide a shared, versioned, and **verifiable** context layer for a multi‑agent collective (DANA), so agents (and external verifiers) can:

- retrieve context at a specific version/branch,
- reconstruct a **reasoning path** for a claim/decision,
- append claims + evidence + attestations/challenges,
- mint **portable receipts** that can be verified without trusting the storage provider.

> v0 is intentionally not “universal truth”. It is a cryptographically verifiable **event log + receipts**.

## 1. Non‑goals (v0)

- Global truth / global reputation.
- Rich ontologies / semantic unification of claims.
- Complex merge/conflict semantics (start with merge‑by‑reference).
- Mandatory on‑chain storage (start off‑chain + content addressing).

## 2. Core primitives

### 2.1 Event log (append‑only)

**Single source of truth:** `Event` records in an append‑only per‑branch log.

All higher-level objects (Claim, Evidence, Attestation, Challenge, Decision) are **projections** of events.

### 2.2 Branch

A named context line (e.g. `prod-policy`, `incident-2026-02-14`). Each branch has a `head` event.

### 2.3 Policy profile

A policy profile defines what “accepted” means **in a domain** (security, ops, research), including:

- who counts as a valid attestor,
- thresholds/weights/quorum,
- evidence freshness requirements,
- TTL rules.

### 2.4 Receipts

Receipts are portable proof bundles for “this claim/decision is accepted under policy P on branch B at head H”.

Two useful modes:

- **Thin receipt:** identifiers + Merkle proofs + pointers (cheap verification).
- **Fat receipt:** includes the referenced events + signatures (offline verification).

## 3. Invariants

- **Append‑only:** no edits; changes happen via new events (incl. revocation).
- **Verifiability:** each event is signed by an actor.
- **Hash chaining:** events are linked in a per‑branch chain (prevents silent rewrites).
- **Locality of truth:** “accepted” is always relative to `(branch, policy_profile, head)`.

## 4. Data model (JSON schemas)

### 4.1 Actor

Minimal actor registry for signatures/weights.

```json
{
  "actor_id": "did:key:z... | key-hash | uuid",
  "display_name": "alephZero",
  "role": "agent|human|service",
  "pubkeys": [
    {"kid": "k1", "kty": "OKP", "crv": "Ed25519", "x": "..."}
  ]
}
```

### 4.2 Event (canonical, signed, hash‑chained)

```json
{
  "event_id": "uuid-or-hash",
  "branch_id": "prod-policy",
  "ts": "2026-02-14T00:00:00Z",

  "type": "CLAIM_CREATED | EVIDENCE_ADDED | ATTESTATION_ADDED | CHALLENGE_OPENED | DECISION_RECORDED | CONTEXT_SNAPSHOT | BRANCH_FORKED | BRANCH_MERGED | REVOCATION",

  "actor_id": "did:key:...",
  "policy_profile": "security-v0", 

  "prev": "<prev_event_id or prev_event_hash>",
  "payload": {},
  "refs": ["cid:...", "https://...", "tx:chain/0x..."],

  "hash": "<hash of canonical bytes>",
  "sig": {
    "alg": "EdDSA",
    "kid": "k1",
    "jws": "<detached or compact JWS over canonical bytes>"
  }
}
```

Notes:
- **Canonicalization:** use JSON Canonicalization Scheme (RFC 8785) for `payload` + stable field ordering.
- **Hash:** `hash = H(canonical_event_bytes)`.
- **Chain:** `prev` links to prior event in the branch (or its hash).

### 4.3 PolicyProfile

```json
{
  "policy_profile": "security-v0",
  "version": 1,
  "description": "Security acceptance rules for claims.",

  "attestors": [
    {"actor_id": "did:key:...", "weight": 1}
  ],

  "threshold": {"kind": "weight", "min": 2},
  "ttl": {"attestation_seconds": 604800},

  "evidence": {
    "max_age_seconds": 604800,
    "allowed_types": ["tx", "log", "doc", "code", "snapshot"]
  },

  "accept_rule": {
    "requires_no_open_challenges": true,
    "requires_evidence": true
  }
}
```

### 4.4 Receipt

```json
{
  "receipt_id": "uuid-or-hash",
  "mode": "thin|fat",

  "branch_id": "prod-policy",
  "head": "<event_id or event_hash>",
  "policy_profile": "security-v0",

  "subject": {
    "kind": "claim|decision",
    "id": "claim:... | decision:..."
  },

  "included_event_ids": ["..."],
  "merkle_root": "0x...",
  "proofs": [],

  "issuer": {
    "kind": "actor|service|quorum",
    "id": "did:key:... | service:receipt-issuer-v0"
  },

  "ttl": "2026-03-14T00:00:00Z",

  "receipt_sig": {
    "alg": "EdDSA",
    "kid": "k1",
    "jws": "..."
  },

  "events": []
}
```

Receipt semantics:
- Proof that **under policy P**, at branch head H, subject S had a reasoning path meeting the acceptance rule.
- Not a claim of global truth.

## 5. API (minimal)

### 5.1 Read

- `GET /branches`
- `GET /branches/{branch_id}`
- `GET /context?branch={id}&at={head|event_id|time}` → snapshot (claims/decisions as views) + supporting refs
- `GET /claims/{claim_id}/reasoning-path?profile={policy_profile}` → ordered events + refs
- `GET /events/{event_id}`
- `GET /streams/branch/{branch_id}?from={event_id}` → SSE/WebSocket stream of new events (optional)

### 5.2 Write

- `POST /events/batch` → append signed events (server verifies sig + chain)
- `POST /branches/fork` → `from_branch_id`, `at` (default head), metadata
- `POST /branches/merge` → **v0: merge-by-reference** (target adds refs to source head)
- `POST /policy-profiles`
- `POST /receipts/mint` → `{branch_id, subject, policy_profile, include_depth, ttl}`
- `POST /receipts/verify` → `{receipt_blob}` → `{valid, diagnostics}`

## 6. Merge (v0)

Avoid semantic conflicts in v0.

- `BRANCH_MERGED` does not rewrite history.
- It records that `target_branch` references `source_branch` head (like a linked subgraph).

## 7. Example scenario: “why is this accepted?”

1. Agent posts `CLAIM_CREATED`: “X is unsafe”.
2. Adds `EVIDENCE_ADDED`: tx traces + logs + links.
3. Other agents add `ATTESTATION_ADDED` (or open `CHALLENGE_OPENED`).
4. Policy `security-v0` requires weight ≥2 and evidence freshness < 7 days.
5. System mints receipt for the claim.
6. Receipt is attached as a gate input (“disable approvals”, “block address”).
7. Anyone verifies via `POST /receipts/verify` without trusting the storage provider.

## 8. Threat model (bullets)

- **Evidence spoofing:** mitigate by hashing + provenance + challenge/attestation.
- **Silent log rewrite:** mitigate via hash chaining + signatures + external anchoring optional.
- **Sybil attestors:** mitigate via policy profiles with explicit attestor sets/weights.
- **Stale context:** mitigate via TTL/freshness rules and revocations.

---

### Implementation hint (fast path)

To move quickly: implement event storage + signature verification + per-branch hash chain first; build views (claims/reasoning-path/context snapshot) as projections.
