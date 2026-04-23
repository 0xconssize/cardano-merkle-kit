# DESIGN.md — Cardano Merkle Accumulator Ecosystem

This file unifies the repository's canonical agent guidance and design memory. It consolidates previous long-form agent and memory documents under `doc/design`.

---

## Purpose

Provide a single source of truth for agent behavior, repository conventions, design decisions, and next actions for the Cardano Merkle Accumulator Ecosystem. This file consolidates long-form design documents and the decision memory used by maintainers and automation.

---

## Project description

An off-chain verification library that provides core primitives for merkleized accumulators on Cardano. The library is composed of two complementary parts:

**On-chain Validation** (Smart Contract Component):
- Implemented by user validators (e.g., via Aiken smart contracts)
- Performs structural consistency checks on the accumulator state (INV-1..7, renumbered after removal of redundant INV-5)
- Validates UTxO chain continuity and metadata integrity
- Does NOT verify IPFS content or cryptographic proofs

**Off-chain Verification** (This Library):
- Provides types, implementations, and utilities for merkleized accumulator handling
- Implements tree-specific proof verification (membership, non-membership)
- Validates binding hashes and CBOR serialization
- Enables construction and verification of accumulator state transitions
- Abstracts over tree types (StandardMerkle, MMR, SparseMerkle, etc.) via trait-based implementations

The library is **pure and composable** (not a smart contract). It provides:
- **Types**: `AccumulatorDescriptor`, `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorObject`
- **Built-in tree implementations**: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry (with trait extensibility)
- **CBOR serialization**: canonical encoding for off-chain accumulator objects
- **Proof verifiers**: tree-specific membership and non-membership proof verification

User validators import the library's on-chain components (invariant checks) and combine them with custom authorization logic. Off-chain applications use the library's verification and tree implementations for proof construction and validation.

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

1. Clear separation of on-chain validation vs off-chain verification

   **On-chain (smart contract):** Validates structural consistency only (INV-1..7). Does not verify IPFS content, proofs, or binding hashes.
   **Off-chain (this library):** Performs all cryptographic verification (proof validation, content hashing, state reconstruction, tree identity anchoring).
   Each `AccumulatorObject` stored in IPFS MUST embed the `prev_tx_ref` (txHash + outputIndex) and `tree_index` from its corresponding datum so that off-chain verifiers can anchor the IPFS object to its on-chain position without traversing the full chain. This is a forward reference from IPFS to Cardano, which is acceptable for anchoring. What MUST NOT appear in the IPFS object is the IPFS CID itself (which would be circular).

2. Abstraction of tree type and hash algorithm

   The protocol treats `tree_type` and `hash_algorithm` as parameters in the `AccumulatorDescriptor`. Supported initial types: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry. Supported algorithms: Blake2b256, Blake2b224, SHA2_256, SHA3_256, Keccak256, Poseidon (Poseidon verification is off-chain).

3. Composability of Datum and Redeemer via tag-based Map protocol

   Within a Map-typed Plutus Data, the integer key `142` points to the `AccumulatorDatum` or a list of them. The same reserved key is used in Redeemers for `AccumulatorAction`. We will use aiken tag capabilities to mark our datum and redeemer with the `142` key.

4. Library provides off-chain verification, not on-chain validation

   This library does NOT implement on-chain validators. Instead, it provides reusable components (types, proof verifiers, CBOR serialization) that smart contract developers use to build their validators. User contracts remain responsible for authorization logic and for composing on-chain invariant checks with authorization.

5. Cardano storage optimization: minimal datum, IPFS for full state, metadata avoids repetition

   Cardano penalizes on-chain storage (UTxO size affects transaction costs; metadata has a 16KB limit). `AccumulatorDatum` stores only essential data needed for on-chain validation (descriptor, root_hash, leaf_count, binding_hash, prev_tx_ref, tree_index). `ipfs_cid` is NOT stored in the datum — it is off-chain only and is published in the producing transaction's metadata (key 674), keyed by `(prev_tx_ref.tx_hash, prev_tx_ref.output_index, tree_index)`. Off-chain verifiers retrieve it by looking up the transaction that created the UTxO. `binding_hash` remains in the datum as the sole on-chain cryptographic commitment to the IPFS content.

---

## Core types (summary)

**AccumulatorDatum** (on-chain, minimal) — stores only fields essential for on-chain validation:
- `descriptor` — tree type, hash algorithm, schema version (immutable per accumulator)
- `root_hash` — current tree root (needed for proof validation)
- `leaf_count` — number of leaves (needed for INV-3 monotonicity check)
- `binding_hash` — cryptographic commitment to the AccumulatorObject in IPFS; `hash_algorithm(CBOR(AccumulatorObject))`; the sole on-chain anchor to off-chain state
- `prev_tx_ref` — `OutputReference { tx_hash: TxHash, output_index: Int }` of the previous UTxO (ensures chain continuity, INV-4); fully identifies the consumed UTxO
- `tree_index` — fixed integer identifying this tree within the UTxO (starts at 0, set once at `InitAccumulator`, never changes); multiple trees may share the same UTxO via tag `142` — `tree_index` distinguishes them (INV-5)

> Note: `ipfs_cid` is intentionally absent from the datum. It is off-chain data published in the producing transaction's metadata and recovered by off-chain verifiers via chain indexing.

**AccumulatorObject** (off-chain, full state) — integer-keyed CBOR map stored permanently in IPFS:
- Key 0: `schema_version` — format version for forward compatibility
- Key 1: `tree_type` — tree implementation identifier
- Key 2: `hash_algorithm` — hash function used
- Key 3: `root_hash` — current root (mirrors datum for verification)
- Key 4: `leaf_count` — current leaf count (mirrors datum for verification)
- Key 5: `leaves` — complete leaf data (tree-specific serialization)
- Key 6: `internal_nodes` — merkle proof nodes (tree-specific serialization)
- Key 7: `prev_tx_hash` — txHash component of `prev_tx_ref` (anchors this IPFS object to its on-chain position; mirrors datum for off-chain verification, VERIF-5)
- Key 8: `prev_output_index` — outputIndex component of `prev_tx_ref` (combined with Key 7 uniquely identifies the consumed UTxO; VERIF-5)
- Key 9: `tree_index` — fixed tree identifier (mirrors datum `tree_index`; enables off-chain identification without full chain traversal; VERIF-5)

**Transaction Metadata** (Cardano standard key 674) — carries the `ipfs_cid` for each tree state and optional indexing hints:
- **`ipfs_cid`** (required per tree) — CID of the new `AccumulatorObject` produced by this transaction; keyed by `(prev_tx_ref.tx_hash, prev_tx_ref.output_index, tree_index)` to unambiguously identify which tree it belongs to
- Descriptor context (optional) — if multiple accumulator actions in the same transaction share the same descriptor, it may be shared here to reduce repetition
- Indexing hints (optional) — labelling which `tree_index` corresponds to which domain object for off-chain applications
- Note: NOT used for storing full AccumulatorObject (respects 16KB limit); see IPFS for complete state

**AccumulatorAction** — user contract actions: InitAccumulator, UpdateAccumulator, MigrateDescriptor, RemoveAccumulator.

---

## On-Chain Validator Invariants (implemented by smart contracts using this library)

These invariants are structural checks enforced on-chain. User contracts call library functions (in `lib/accumulator/invariants.ak`) to validate these properties:

INV-1: `descriptor.tree_type` must not change between prev and new datum (except via `MigrateDescriptor`).

INV-2: `descriptor.hash_algorithm` must not change (except via `MigrateDescriptor`).

INV-3: `new_leaf_count >= prev_leaf_count` (no implicit shrinking).

INV-4: `new.prev_tx_ref` must equal the `OutputReference` of the consumed input (ensures chain continuity).

INV-5: `tree_index` must not change across state transitions — `new_tree_index == prev_tree_index`. The tree_index is set once at `InitAccumulator` and is immutable for the lifetime of the tree.

INV-6: `len(binding_hash)` must match expected length for `hash_algorithm`.

INV-7: `len(root_hash)` must match expected length for `hash_algorithm`.

## Off-Chain Verification Checks (performed by this library)

VERIF-1: `binding_hash == hash_algorithm(CBOR(AccumulatorObject))` — Content integrity validation. `ipfs_cid` is obtained from the producing transaction's metadata; `binding_hash` from the datum is the on-chain anchor used to verify the fetched content.

VERIF-2: Proof verification — tree-specific membership/non-membership proof validation via `ProofVerifier` trait.

VERIF-3: CBOR canonical encoding — all off-chain objects must serialize and deserialize to canonical form.

VERIF-4: Accumulated state reconstruction — replay accumulator transitions to validate historical consistency and state lineage.

VERIF-5: IPFS anchoring consistency — `AccumulatorObject.prev_tx_hash` must equal `datum.prev_tx_ref.tx_hash`, `AccumulatorObject.prev_output_index` must equal `datum.prev_tx_ref.output_index`, and `AccumulatorObject.tree_index` must equal `datum.tree_index`. This ensures the IPFS object is correctly bound to its on-chain position (identified by `prev_tx_ref + tree_index`) and cannot be swapped across trees or chain positions.

---

## Decisions and rationale (memory)

D-01: Generic abstraction for tree types — choose appropriate algorithm per use case (MMR for append-only, Sparse for non-membership).

D-02: IPFS + Filecoin for addressability and durability respectively; `ipfs_cid` published in transaction metadata (key 674), NOT stored in datum (off-chain only).

D-03: Unidirectional binding hash (Cardano -> IPFS) with controlled back-reference for anchoring. The binding_hash in the datum commits to the AccumulatorObject content (which includes prev_tx_ref fields and tree_index). The AccumulatorObject MUST NOT contain the IPFS CID of itself (that would be circular, since the CID is derived from the content). However, including Cardano on-chain references (`prev_tx_hash`, `prev_output_index`, `tree_index`) in the AccumulatorObject is intentional and necessary for off-chain tree identification.

D-04: Tree identity = `(genesis_tx_hash, genesis_output_index, tree_index)`. Each tree's genesis is the `InitAccumulator` transaction output. `prev_tx_ref` (a Cardano `OutputReference = { tx_hash, output_index }`) chains each UTxO back to its predecessor; following the chain leads to the genesis. No separate `output_index` field is stored in the datum — the successor transaction's off-chain builder resolves the current UTxO's `OutputReference` from the blockchain directly. `tree_index` is a fixed integer assigned at genesis distinguishing multiple trees within the same UTxO.

D-05: Filecoin deals as primary persistence; optional `filecoin_deal_id` in datums.

D-06: Datum composability using tag `142` for accumulator (Aiken `@tag` annotation). User validators extract via type annotation.

D-07: Redeemer composability using tag `142` for `AccumulatorAction`.

D-08: Library scope — this is an off-chain verification library, not a smart contract. It provides reusable types, proof verifiers, and CBOR utilities. Smart contracts using the library implement on-chain invariant checks (INV-1..7) and authorization logic. Off-chain applications use the library for proof construction, binding hash validation, and state transition verification.

D-09: Trait-based tree implementations — each tree type (built-in or custom) implements a `ProofVerifier` interface. Built-in trees included in library; custom trees can be user-provided.

D-10: Built-in trees (StandardMerkle, MMR, SparseMerkle, MerklePatriciaForestry) live in the library repo. Off-chain integration tools (IPFS client, proof constructor, verifier) are optional, added in follow-up phases.

D-11: Transaction metadata strategy — `AccumulatorDatum` contains 6 on-chain validation essentials (descriptor, root_hash, leaf_count, binding_hash, prev_tx_ref, tree_index). `ipfs_cid` is absent from the datum: it is off-chain data published in the producing transaction's metadata (key 674), keyed by `(prev_tx_ref.tx_hash, prev_tx_ref.output_index, tree_index)`. `binding_hash` in the datum is the sole on-chain cryptographic commitment to the IPFS content. Off-chain validation: (1) find the tx that produced the UTxO via chain indexing, (2) read `ipfs_cid` from its metadata keyed by `(prev_tx_ref, tree_index)`, (3) fetch AccumulatorObject from IPFS, (4) verify IPFS anchoring (VERIF-5), (5) verify `binding_hash` (VERIF-1), (6) apply tree proof validation (VERIF-2), (7) reconstruct cumulative state by replaying subsequent on-chain transactions.

D-12: Tree identification and multi-tree UTxOs — a single UTxO may carry multiple `AccumulatorDatum` entries via the tag `142` composability (a list of datums). Each entry is distinguished by `tree_index` (an integer starting at 0). The tree_index is fixed at `InitAccumulator` and validated as immutable by INV-5 on every subsequent transition. The composite key `(prev_tx_ref.tx_hash, prev_tx_ref.output_index, tree_index)` uniquely identifies a tree at any given point in its history; the globally unique genesis identity `(genesis_tx_hash, genesis_output_index, tree_index)` is recoverable by following the `prev_tx_ref` chain to `InitAccumulator`. Each IPFS AccumulatorObject embeds `prev_tx_hash`, `prev_output_index`, and `tree_index` (keys 7–9) so that off-chain verifiers and metadata parsers can identify the tree without full chain traversal.

---

## Library Architecture: On-Chain & Off-Chain Components

### Data Storage Strategy: Cardano-Optimized Separation

**Principle:** Minimize on-chain data (expensive UTxO storage) while preserving complete verifiability off-chain.

| Data | Location | Size | On-Chain Cost | Rationale |
|------|----------|------|---------------|-----------|
| `descriptor` | UTxO datum | ~3 bytes | High (essential) | Immutable config; needed for validator routing |
| `root_hash` | UTxO datum | 32-64 bytes | High (essential) | Required for on-chain invariant validation |
| `leaf_count` | UTxO datum | ~8 bytes | High (essential) | Required for monotonicity check (INV-3) |
| `binding_hash` | UTxO datum | 32-64 bytes | High (essential) | Sole on-chain commitment to IPFS content; prevents tampering |
| `prev_tx_ref` | UTxO datum | ~34 bytes | High (essential) | `OutputReference { tx_hash, output_index }` of consumed UTxO; chain continuity INV-4 |
| `tree_index` | UTxO datum | ~1 byte | High (essential) | Fixed per-tree id within UTxO; distinguishes multiple trees; INV-5 |
| **Total Datum Size** | **UTxO** | **~145 bytes** | **Minimal** | Only validation-critical fields |
| `ipfs_cid` | Transaction metadata (674) | ~36 bytes | Low | Off-chain only; keyed by `(prev_tx_ref, tree_index)`; not needed on-chain |
| Full `AccumulatorObject` | IPFS | ~1-100 KB | Free (off-chain) | Complete tree state + on-chain anchors (keys 7–9); permanent storage via Filecoin |

### On-Chain Components (for Smart Contract Developers)

The library provides **on-chain invariant checking**:
1. **Types** — `AccumulatorDescriptor`, `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorObject` (CBOR schema), serializable to Plutus Data with tag `142`.
2. **Invariant validators** (in `lib/accumulator/invariants.ak`) — functions to check INV-1..7 (descriptor stability, leaf count monotonicity, tree_index immutability, hash length validation, prev_tx_ref continuity).
3. **Type utilities** — helper functions for type coercion, datum extraction, and action validation.

**User Contracts Implement**:
- Authorization logic (who may update).
- Binding hash composition and validation (off-chain computed, on-chain stored).
- On-chain integration with other protocols or state machines.

### Off-Chain Components (for Verification & Proof Construction)

The library provides **off-chain verification utilities**:
1. **CBOR serialization** — canonical encode/decode of `AccumulatorObject` and related structures (per Cardano standards).
2. **Proof verifiers** (trait-based) — tree-specific membership/non-membership proof validation.
3. **Binding hash utilities** — compute and validate cryptographic commitments to off-chain state (used to validate IPFS content).
4. **IPFS integration** — resolve `ipfs_cid` from the producing transaction's metadata (key 674, keyed by `(prev_tx_ref, tree_index)`); fetch `AccumulatorObject` from IPFS; validate against `binding_hash` from datum (VERIF-1) and verify embedded anchors (VERIF-5).
5. **State reconstruction** — reconstruct accumulator state by: (a) finding the tx that produced the UTxO via chain indexing, (b) reading `ipfs_cid` from its metadata keyed by `(prev_tx_ref, tree_index)`, (c) fetching `AccumulatorObject` from IPFS, (d) verifying IPFS anchors (VERIF-5) and `binding_hash` (VERIF-1), (e) replaying subsequent on-chain transactions (redeemer + new datum), (f) validating each transition including INV-5.
6. **Transaction metadata handling** — parse optional Cardano transaction metadata (key 674) for descriptor-sharing hints and off-chain indexing tags within a transaction batch.

**Built-in Tree Implementations**:
- `lib/trees/standard_merkle.ak` — StandardMerkle proofs (membership only).
- `lib/trees/merkle_mmr.ak` — Merkle-based MMR proofs (append-only).
- `lib/trees/sparse_merkle.ak` — SparseMerkle proofs (membership + non-membership).
- `lib/trees/merkle_patricia_forestry.ak` — MerklePatriciaForestry proofs (complex structures).

Each implements the `ProofVerifier` trait:
```aiken
pub type ProofVerifier {
  verify_membership(root: ByteArray, leaf: Data, proof: Data) -> Bool
  verify_non_membership(root: ByteArray, element: Data, proof: Data) -> Bool
}
```

**Custom Tree Implementations**: Users can extend the library by implementing `ProofVerifier` for domain-specific tree types.

### Integration Patterns

**On-Chain (Smart Contract Example)**:
```aiken
import lib/accumulator/types
import lib/accumulator/invariants
import lib/trees/standard_merkle

validator my_contract(datum: Data, redeemer: Data, ctx: ScriptContext) {
  let accumulator: AccumulatorDatum = datum     // Aiken extracts tag 142
  let action: AccumulatorAction = redeemer      // Aiken extracts tag 142
  
  // On-chain: check structural invariants only
  invariants.validate_all(prev_datum.accumulator, accumulator, ctx)
  && my_authorization_logic(ctx)
}
```

**Off-Chain (Proof Verification Example)**:
```typescript
// Fetch from IPFS
const accumulatorObject = await ipfs.get(datum.ipfs_cid);

// Off-chain verification
const isValid = 
  verifyBindingHash(accumulatorObject, datum.binding_hash, datum.descriptor.hash_algorithm) &&
  StandardMerkle.verifyMembership(datum.root_hash, leaf, proof);
```

---

## Next recommended implementation steps

### Phase 1: Core Library (On-Chain & Off-Chain Shared)

1. Initialize Aiken project and `aiken.toml`.
2. Implement `lib/accumulator/types.ak` (all core types).
3. Implement `lib/accumulator/cbor.ak` for canonical CBOR serialization/deserialization.
4. Implement `lib/accumulator/invariants.ak` with INV-1..INV-7 validators and unit tests.

### Phase 2: Built-in Tree Implementations (Off-Chain Verification)

5. Implement `lib/trees/standard_merkle.ak` (reference implementation, membership proof).
6. Implement `lib/trees/merkle_mmr.ak` (append-only accumulator).
7. Implement `lib/trees/sparse_merkle.ak` (membership + non-membership proofs).
8. Implement `lib/trees/merkle_patricia_forestry.ak` (complex tree structures).

### Phase 3: Documentation & Examples (On-Chain Integration)

9. Create example validator in `examples/example_validator.ak` (demonstrates on-chain invariant checking + authorization).
10. Document binding hash computation and validation strategy.
11. Property-based tests for tree implementations and invariant validators.

### Phase 4: Off-Chain Tooling (Future)

12. (Future) TypeScript/Rust off-chain verification library: AccumulatorObject builder, binding hash computation, IPFS CID management, proof validation utilities.

---

## Open questions (to decide later)

- Q-01: Custom tree test case — define a minimal custom tree type to validate trait abstraction (e.g., a 2-level tree for testing).
- Q-02: Example authorization model for sample validator (PubKeyHash, NFT, multisig, or simple demo).
- Q-03: Whether to include precomputed proofs in the IPFS object as an optional field.
- Q-04: Formalize schema as a CIP after stabilization.
- Q-05: Off-chain phase 2 — priority for TypeScript tools (proof constructor, verifier, history API).
- Q-06: Transaction metadata schema (key 674) — define the exact CBOR structure for entries keyed by `(prev_tx_ref.tx_hash, prev_tx_ref.output_index, tree_index)` containing `ipfs_cid` and optional descriptor/indexing hints. Should the composite key be a CBOR list or a nested map?
- Q-07: State reconstruction performance — for chains with many accumulator update transactions, should off-chain indexing cache intermediate AccumulatorObject snapshots at checkpoints, or always reconstruct from the nearest IPFS snapshot + transaction replay?
- Q-08: `tree_index` assignment order — when `InitAccumulator` creates multiple trees in one transaction (tag `142` list), is tree_index assigned by the order of datums in the list (0, 1, 2...) or by another convention? Recommend: positional order within the tag `142` list, validated off-chain.

---

## References

This file is the canonical reference for agents and decisions; previous long-form docs were consolidated here.
