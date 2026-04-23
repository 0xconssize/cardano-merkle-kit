# DESIGN.md — Cardano Merkle Accumulator Ecosystem

This file unifies the repository's canonical agent guidance and design memory. It consolidates previous long-form agent and memory documents under `doc/design`.

---

## Purpose

Provide a single source of truth for agent behavior, repository conventions, design decisions, and next actions for the Cardano Merkle Accumulator Ecosystem. This file consolidates long-form design documents and the decision memory used by maintainers and automation.

---

## Project description

An Aiken library that provides core primitives for merkleized accumulators on Cardano. The library enables any contract to link an off-chain accumulator (Merkle tree variants) stored in IPFS/Filecoin with an on-chain digest anchored in an eUTxO, using trait-based tree implementations.

The project is a **pure library** (not an example validator). It provides:
- **Types**: `AccumulatorDescriptor`, `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorObject`
- **Invariant validators**: on-chain structural checks (INV-1..7)
- **CBOR serialization**: canonical encoding for off-chain accumulator objects
- **Built-in tree implementations**: StandardMerkle, MerkleMMR, SparseMerkle, SparsePatriciaForestRei (and extensibility for custom trees)

User validators import the library and provide authorization logic + tree-specific proof verification.

---

## Stack

- Blockchain: Cardano (mainnet/preprod)
- Smart contracts: Aiken (compiles to UPLC / Plutus Core)
- Plutus target: Plutus V3
- Off-chain storage: IPFS (CIDv1) + Filecoin
- Serialization: CBOR canonical encoding
- Off-chain tooling: TypeScript (EffectTs) and Rust with WASM implementation
- Testing: Aiken tests + property-based tests where applicable

---

## Non-negotiable design principles

1. Separation of responsibilities on-chain / off-chain

   The IPFS object is a static commitment to the tree and must not contain references to Cardano transactions. Temporal and causal traceability lives on-chain via `prev_tx_ref` chains.

2. Abstraction of tree type and hash algorithm

   The protocol treats `tree_type` and `hash_algorithm` as parameters in the `AccumulatorDescriptor`. Supported initial types: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry. Supported algorithms: Blake2b256, Blake2b224, SHA2_256, SHA3_256, Keccak256, Poseidon (Poseidon verification is off-chain initially).

3. Composability of Datum and Redeemer via tag-based Map protocol

   Within a Map-typed Plutus Data, the integer key `142` points to the `AccumulatorDatum` or a list of them. The same reserved key is used in Redeemers for `AccumulatorAction`. We will use aiken tag capabilities to mark our datum and redeemer with the `142` key.

4. Validator verifies structural consistency, not IPFS content

   On-chain checks are limited to transition invariants, slot validity, and binding/root hash lengths. Full content verification (binding_hash == hash(CBOR(AccumulatorObject))) is performed off-chain.

---

## Core types (summary)

`AccumulatorDescriptor` — contains `tree_type`, `hash_algorithm`, `schema_version`.

`AccumulatorDatum` — includes `descriptor`, `root_hash`, `leaf_count`, `ipfs_cid`, `binding_hash`, `prev_tx_ref`, `output_index`.

`AccumulatorAction` — InitAccumulator, UpdateAccumulator, MigrateDescriptor, RemoveAccumulator.

AccumulatorObject (CBOR schema) — integer-keyed map using compact keys (0..6) describing schema_version, tree_type, hash_algorithm, root_hash, leaf_count, leaves, and internal nodes.

---

## Validator invariants (to implement in `validate.ak`)

INV-1: `descriptor.tree_type` must not change between prev and new datum (except via `MigrateDescriptor`).

INV-2: `descriptor.hash_algorithm` must not change (except via `MigrateDescriptor`).

INV-3: `new_leaf_count >= prev_leaf_count` (no implicit shrinking).

INV-4: `new.prev_tx_ref` must equal the `OutputReference` of the consumed input (ensures chain continuity).

INV-5: `output_index` in datum must match the index of the UTxO being spent.

INV-6: `len(binding_hash)` must match expected length for `hash_algorithm`.

INV-7: `len(root_hash)` must match expected length for `hash_algorithm`.

---

## Decisions and rationale (memory)

D-01: Generic abstraction for tree types — choose appropriate algorithm per use case (MMR for append-only, Sparse for non-membership).

D-02: IPFS + Filecoin for addressability and durability respectively; `ipfs_cid` stored on-chain, `filecoin_deal_id` optional.

D-03: Unidirectional binding hash (Cardano -> IPFS) to avoid cryptographic circularity.

D-04: `prev_tx_ref` chain + `output_index` for UTxO identification and temporal traceability.

D-05: Filecoin deals as primary persistence; optional `filecoin_deal_id` in datums.

D-06: Datum composability using tag `142` for accumulator (Aiken `@tag` annotation). User validators extract via type annotation.

D-07: Redeemer composability using tag `142` for `AccumulatorAction`.

D-08: Library model — core library provides types, invariants, and CBOR serialization. Authorization and proof verification are user validator responsibilities.

D-09: Trait-based tree implementations — each tree type (built-in or custom) implements a `ProofVerifier` interface. Built-in trees included in library; custom trees can be user-provided.

D-10: Built-in trees (StandardMerkle, MMR, SparseMerkle, SparsePatriciaForestRei) live in the library repo. Off-chain integration tools (IPFS client, proof constructor, verifier) are optional, added in follow-up phases.

---

## Library Architecture & Tree Implementation Pattern

### Core Library Scope
The library provides **three responsibilities**:
1. **Types** — `AccumulatorDescriptor`, `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorObject` (CBOR schema).
2. **Invariant validators** — functions to check INV-1..7 (descriptor stability, leaf count monotonicity, hash length validation).
3. **CBOR serialization** — encode/decode `AccumulatorObject` in canonical form.

**Out of scope** (user validator responsibility):
- Authorization logic (who may update).
- Proof verification (tree-specific; via trait implementation).
- Business logic (domain-specific rules).

### Tree Implementation Pattern
Each tree type implements a `ProofVerifier` trait:
```aiken
pub type ProofVerifier {
  verify_membership(root: ByteArray, leaf: Data, proof: Data) -> Bool
  verify_non_membership(root: ByteArray, element: Data, proof: Data) -> Bool
  // Additional tree-specific operations as needed
}
```

**Built-in implementations** in library:
- `lib/trees/standard_merkle.ak`
- `lib/trees/merkle_mmr.ak`
- `lib/trees/sparse_merkle.ak`
- `lib/trees/sparse_patricia_forestrei.ak`

**Custom trees**: User validators can implement `ProofVerifier` for domain-specific tree types.

### User Validator Pattern
A user contract using the library:
```aiken
import lib/accumulator/types
import lib/accumulator/invariants
import lib/trees/standard_merkle  // or custom tree

validator my_contract(datum: Data, redeemer: Data, ctx: ScriptContext) {
  // Extract via @tag annotation
  let accumulator: AccumulatorDatum = datum  // Aiken extracts tag 142
  let action: AccumulatorAction = redeemer    // Aiken extracts tag 142
  
  // Check invariants
  invariants.validate_all(prev_datum.accumulator, accumulator, ctx)
  && standard_merkle.verify_membership(root, leaf, proof)
  && my_authorization_logic(ctx)
}
```

---

## Next recommended implementation steps

1. Initialize Aiken project and `aiken.toml`.
2. Implement `lib/accumulator/types.ak` (all core types).
3. Implement `lib/accumulator/invariants.ak` with INV-1..INV-7 validators and unit tests.
4. Implement `lib/accumulator/cbor.ak` for canonical CBOR serialization/deserialization.
5. Implement `lib/trees/standard_merkle.ak` (reference implementation, fully tested).
6. Implement `lib/trees/merkle_mmr.ak`.
7. Implement `lib/trees/sparse_merkle.ak`.
8. Implement `lib/trees/sparse_patricia_forestrei.ak`.
9. Create example validator in `examples/example_validator.ak` (demonstrates library usage with authorization).
10. Property-based tests for tree implementations.
11. (Future) Off-chain TypeScript tools: AccumulatorObject builder, binding hash computation, IPFS CID management.

---

## Open questions (to decide later)

- Q-01: Custom tree test case — define a minimal custom tree type to validate trait abstraction (e.g., a 2-level tree for testing).
- Q-02: Example authorization model for sample validator (PubKeyHash, NFT, multisig, or simple demo).
- Q-03: Whether to include precomputed proofs in the IPFS object as an optional field.
- Q-04: Formalize schema as a CIP after stabilization.
- Q-05: Off-chain phase 2 — priority for TypeScript tools (proof constructor, verifier, history API).

---

## References

This file is the canonical reference for agents and decisions; previous long-form docs were consolidated here.
