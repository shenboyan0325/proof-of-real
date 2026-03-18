## Proof of Real (C2PA) — EAS Schema Specification (v1)

This document defines the **Proof of Real** attestation schema: a minimal, composable way to publish **verifiable “real-world capture” claims** for media that carries **C2PA** provenance, using **EAS (Ethereum Attestation Service)**.

The intended use is to support:
- AI/agent data pipelines (trustable inputs, provenance checks)
- media authenticity workflows (fact-checking, newsroom verification, compliance)
- marketplaces for verified real-world datasets

This is a **schema + verification convention**, not an application.

---

## 1. Terminology

- **EAS**: Ethereum Attestation Service.
- **Schema**: A typed field layout registered in EAS `SchemaRegistry`, producing a `schemaUID`.
- **Attestation**: A single proof record referencing a `schemaUID`, producing an attestation `uid`.
- **Issuer / Attester**: The address that creates attestations. This spec assumes a canonical “official issuer” model, but can be extended to multiple issuers.

---

## 2. Threat model (what this proves / does not prove)

What it aims to prove:
- A specific media file (identified by `mediaHash`) was checked by an issuer using a defined verification process.
- The media carried C2PA material that passed issuer checks (captured via `c2paManifestHash` and `signerCertHash`).
- Optional metadata (time/location/device) is provided in a privacy-aware way (hashed or coarse numeric encoding).

What it does *not* automatically prove:
- Ground-truth that the scene is “real” in a philosophical sense. It proves **an issuer’s verifiable statement** and the evidence it binds to.
- That the issuer is trustworthy. Consumers must decide which issuers they accept.
- That the media bytes served later match the original unless the verifier recomputes `mediaHash`.

---

## 3. Chain deployments (canonical + mirrors)

This spec locks **Optimism** as the **canonical chain** for the “official” schema publication. Mirror deployments on other chains are optional and should be added only when needed.

### 3.1 Canonical chain (Optimism)

- **Network**: Optimism (OP Mainnet)
- **SchemaRegistry**: `0x4200000000000000000000000000000000000020`
- **EAS**: `0x4200000000000000000000000000000000000021`
- **Schema UID (v1)**: `0xa4e63db52bff5b292538d117bfee61a4282bd5f0aa8a112b29bed2af077c6e87`
- **Schema page**: `https://optimism.easscan.org/schema/view/0xa4e63db52bff5b292538d117bfee61a4282bd5f0aa8a112b29bed2af077c6e87`

### 3.2 Optional mirrors (not used for now)

This project currently publishes **only on Optimism**. If you later mirror to other chains, keep the **same schema string** and publish a mapping table in this section.

---

## 4. Roles

### 4.1 Official issuer model (recommended for v1)

- **Official Issuer / Attester**: `0xEaaB4F46b56F4Ba892231a65Da4E05EC2f40386C`

Verification clients SHOULD treat attestations as “official Proof of Real” only if:
- `attester == OfficialIssuer`, and
- the attestation is not revoked, and
- `verificationScoreBps` meets the client’s policy threshold.

You may later extend to multiple issuers by publishing an issuer allowlist (offchain, or onchain via a separate registry schema).

#### Issuer key management & rotation (normative guidance)

- The issuer key MAY be a hot wallet (it does not need to custody large assets), but it MUST be treated as a **brand signing key**. Compromise enables fraudulent attestations that harm trust.
- The issuer address CAN be rotated, but historical attestations are immutable:
  - Existing attestations will always show the original `attester` and cannot be “edited”.
  - Rotation is performed by publishing a new accepted issuer and having clients update policy.

Clients SHOULD implement an allowlist with time bounds:
- `issuer`: `0x...`
- `effectiveFrom`: (block number or timestamp)
- optional `effectiveUntil`: (block number or timestamp)

Issuers SHOULD announce rotations publicly and provide a transition window where both old and new issuers are accepted.

---

## 5. Canonical schema (v1, standard fields)

### 5.1 Schema string

Use this exact schema string in EAS:

- `bytes32 mediaHash`
- `bytes32 c2paManifestHash`
- `bytes32 signerCertHash`
- `uint64 captureTimestamp`
- `int32 captureLatE7`
- `int32 captureLonE7`
- `bytes32 captureDeviceHash`
- `uint16 verificationScoreBps`
- `bytes32 statementURIHash`
- `bytes32 refMediaHash`

### 5.2 Field definitions

- **mediaHash** (`bytes32`)
  - Hash of the exact media bytes verified by the issuer.
  - Recommended: `keccak256(fileBytes)`.

- **c2paManifestHash** (`bytes32`)
  - Hash that represents the C2PA manifest (or a stable digest extracted from it).
  - Recommended: `keccak256(c2paManifestBytes)` or a fixed digest computed from normalized fields.

- **signerCertHash** (`bytes32`)
  - Hash identifying the C2PA signer certificate (privacy-preserving).
  - Recommended: `keccak256(certDerBytes)` (or fingerprint of the leaf cert).

- **captureTimestamp** (`uint64`)
  - Unix timestamp in seconds, derived from C2PA where available.
  - Use `0` if unknown.

- **captureLatE7 / captureLonE7** (`int32`, `int32`)
  - Latitude/longitude multiplied by 1e7 (E7 encoding) and rounded to integer.
  - Use `0` when unknown or intentionally omitted for privacy.
  - Clients SHOULD treat (0,0) as “unknown” (not as real coordinates).

- **captureDeviceHash** (`bytes32`)
  - Hash of a device descriptor string or identifier (privacy-preserving).
  - Recommended: `keccak256(utf8(deviceDescriptor))`.

- **verificationScoreBps** (`uint16`)
  - Score in basis points, range 0–10000.
  - This is issuer-defined but MUST be documented (see section 6).

- **statementURIHash** (`bytes32`)
  - Hash pointer to an offchain verification statement/report (JSON).
  - Recommended: `keccak256(utf8(uri))` (stable & cheap), or `keccak256(reportBytes)` (stronger binding).
  - The chosen method MUST be declared and kept consistent.

- **refMediaHash** (`bytes32`)
  - Optional reference to an “origin media” hash (e.g., edited version referencing an original).
  - Use `0x0...0` when not applicable.

---

## 6. Verification process (normative)

Issuers MUST:
- Extract and validate C2PA provenance using a well-defined implementation.
- Compute `mediaHash` deterministically from the exact bytes they verified.
- Produce an offchain verification statement (JSON) with:
  - spec version (`proofOfRealSpecVersion`)
  - hashing algorithm identifiers
  - C2PA verification results (pass/fail + reasons)
  - extracted timestamps/claims used
  - software version / build identifier
  - issuer signature (optional but recommended)

Clients SHOULD verify:
- attestation is present under the canonical `schemaUID`
- attestation is not revoked
- attester is an accepted issuer
- `verificationScoreBps` meets policy threshold
- optional: fetch statement and verify its integrity according to the published `statementURIHash` rule

---

## 7. Scoring policy (recommended baseline)

To keep interoperability, publish a simple baseline scoring rubric (example):
- Start at 10000
- Subtract points for:
  - missing timestamp / location
  - weak device identity
  - partial C2PA chain validation
  - unsupported signing chain

Important: the exact rubric is up to the issuer, but the rubric MUST be public if you want third parties to rely on the score.

---

## 8. Privacy considerations

This schema is designed to avoid leaking sensitive data:
- certificates and device identifiers are hashed
- location is numeric and can be zeroed or coarse-grained before encoding

Issuers SHOULD:
- allow “privacy mode” where location/device are omitted (set to 0 / bytes32(0))
- avoid storing personally identifying information in the offchain statement unless necessary

---

## 9. Versioning

Schema changes that affect field list/types MUST be released as a **new schemaUID** (e.g., `v2`).
Do not “silently” change hashing rules or statement integrity rules under the same version.

---

## 10. Reference implementation pointers (optional)

Recommended components:
- C2PA verification: `c2pa` JS/Python tooling
- EAS issuance: `@ethereum-attestation-service/eas-sdk`
- Storage: IPFS / Cloudflare R2

