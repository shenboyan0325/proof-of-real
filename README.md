# proof-of-real

**Proof of Real (C2PA)** is an EAS schema + verification convention to publish verifiable “real-world capture” claims for media that carries **C2PA** provenance.

This repository publishes the canonical schema on **Optimism (OP Mainnet)** and the specification for issuers and verifiers.

## Why
- **AI/agent data pipelines**: trustable inputs & provenance checks
- **Media authenticity workflows**: fact-checking, newsroom verification, compliance
- **Verified dataset markets**: standardized, composable proofs

> This is a **schema + verification convention**, not an application.

## Canonical deployment (Optimism)
- **Network**: Optimism (OP Mainnet)
- **SchemaRegistry**: `0x4200000000000000000000000000000000000020`
- **EAS**: `0x4200000000000000000000000000000000000021`
- **Schema UID (v1)**: `0xa4e63db52bff5b292538d117bfee61a4282bd5f0aa8a112b29bed2af077c6e87`
- **Schema page**: `https://optimism.easscan.org/schema/view/0xa4e63db52bff5b292538d117bfee61a4282bd5f0aa8a112b29bed2af077c6e87`

## Official issuer (v1)
- **Official Issuer / Attester**: `0xEaaB4F46b56F4Ba892231a65Da4E05EC2f40386C`

Verifiers SHOULD treat an attestation as “official” only if:
- it references the canonical `schemaUID`,
- it is not revoked, and
- `attester == Official Issuer`, and
- `verificationScoreBps` meets the verifier’s policy threshold.

## Schema (v1, standard fields)
Schema string:

`bytes32 mediaHash,bytes32 c2paManifestHash,bytes32 signerCertHash,uint64 captureTimestamp,int32 captureLatE7,int32 captureLonE7,bytes32 captureDeviceHash,uint16 verificationScoreBps,bytes32 statementURIHash,bytes32 refMediaHash`

Field highlights:
- `mediaHash`: recommended `keccak256(fileBytes)`
- `verificationScoreBps`: 0–10000 basis points
- `statementURIHash`: hash pointer to an offchain verification statement/report

## Spec
- **Specification**: `./Proof of Real (C2PA) — EAS Schema Specification.md`

## License
MIT
