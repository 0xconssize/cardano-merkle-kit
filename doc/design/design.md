# DESIGN.md — Cardano Accumulator SDK

This file unifies the repository's canonical agent guidance and design memory. It consolidates previous long-form agent and memory documents under `doc/design`.

---

## Purpose

Provide a single source of truth for agent behavior, repository conventions, design decisions, and next actions for the Cardano Accumulator SDK. This file consolidates long-form design documents and the decision memory used by maintainers and automation.

---

## Project description

An off-chain verification library that provides core primitives for merkleized accumulators on Cardano. The library is composed of two complementary parts:

**On-chain Validation** (Smart Contract Component):
- Implemented by user validators (e.g., via Aiken smart contracts)
- Performs structural consistency checks on the accumulator state (INV-4, INV-6, INV-7, INV-8)
- Validates that `datum.policy_id` and `datum.token_name` match the `TreeLifecycleToken` `AssetClass` present in the UTxO (INV-4)
- Chain continuity is guaranteed by the one-shot token (no INV-3 needed)
- Does NOT verify IPFS content, binding hashes, or cryptographic proofs
- `descriptor` and `binding_hash` are intentionally absent from the datum — descriptor is fixed at genesis (minting tx metadata) and IPFS integrity is guaranteed by content addressing

**Off-chain Verification** (This Library):
- Provides types, implementations, and utilities for merkleized accumulator handling
- Implements tree-specific proof verification (membership, non-membership)
- Resolves tree descriptor from the genesis (minting) transaction metadata
- Validates IPFS content integrity via CID (content-addressed; no on-chain binding hash required)
- Enables construction and verification of accumulator state transitions
- Abstracts over tree types (StandardMerkle, MMR, SparseMerkle, etc.) via trait-based implementations

The library is **pure and composable** (not a smart contract). It provides:
- **Types**: `AccumulatorDescriptor`, `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorObject`
- **Built-in tree implementations**: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry (with trait extensibility)
- **CBOR serialization**: canonical encoding for off-chain accumulator objects
- **Proof verifiers**: tree-specific membership and non-membership proof verification

> `AccumulatorDescriptor` is resolved off-chain from the genesis (minting) transaction metadata; it is not stored in the on-chain datum.

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

   **On-chain (smart contract):** Validates that `datum.policy_id` and `datum.token_name` together match the `AssetClass` of the `TreeLifecycleToken` present in the UTxO (INV-4), `root_hash` length (INV-6), and token presence/burn (INV-7, INV-8). Storing both fields prevents an attacker from substituting a token with the same asset name minted under a different policy. Chain continuity is guaranteed by the one-shot token. Does NOT verify IPFS content, proofs, descriptor fields, or binding hashes. `descriptor`, `binding_hash`, `prev_tx_ref`, and `tree_index` are all absent from the datum by design.
   **Off-chain (this library):** Performs all cryptographic verification (proof validation, IPFS content integrity via CID, state reconstruction, tree identity anchoring). Resolves the `AccumulatorDescriptor` by looking up the genesis (minting) transaction's metadata.
   Each `AccumulatorObject` stored in IPFS MUST embed the `token_name` of its `TreeLifecycleToken` so that off-chain verifiers can identify which tree it belongs to (VERIF-5). What MUST NOT appear in the IPFS object is the IPFS CID itself (circular).

2. Abstraction of tree type and hash algorithm

   The protocol treats `tree_type` and `hash_algorithm` as parameters in the `AccumulatorDescriptor`. Supported initial types: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry. Supported algorithms: Blake2b256, Blake2b224, SHA2_256, SHA3_256, Keccak256, Poseidon (Poseidon verification is off-chain).

   The `AccumulatorDescriptor` is published once in the genesis (minting) transaction's metadata (key 674) and is immutable for the lifetime of the tree. It is resolved off-chain by looking up the `TreeLifecycleToken`'s minting transaction.

3. Composability of Datum and Redeemer via Plutus Data constructor tag `142`

   Tag `142` is the Plutus Data constructor index assigned to `AccumulatorDatum` and `AccumulatorAction` via Aiken's `@tag(142)` annotation. When either type is encoded as Plutus Data, constructor index 142 is used. This allows user validators to safely embed these types inside larger datum or redeemer structures and pattern-match on them by constructor index without conflicts with other application-defined constructors.

4. Library provides off-chain verification, not on-chain validation

   This library does NOT implement on-chain validators. Instead, it provides reusable components (types, proof verifiers, CBOR serialization) that smart contract developers use to build their validators. User contracts remain responsible for authorization logic and for composing on-chain invariant checks with authorization.

5. Cardano storage optimization: minimal datum, IPFS for full state, genesis metadata for descriptor

   Cardano penalizes on-chain storage (UTxO size affects transaction costs; metadata has a 16KB limit). `AccumulatorDatum` stores only the three fields required for on-chain validation: `root_hash`, `policy_id`, and `token_name`. `descriptor`, `binding_hash`, `prev_tx_ref`, and `tree_index` are intentionally absent: the descriptor is fixed at genesis and published once in the minting transaction's metadata (key 674); IPFS content integrity is guaranteed by content addressing; `prev_tx_ref` and `tree_index` are no longer needed since `AssetClass(policy_id, token_name)` is the complete, unique tree identity. Each spend transaction publishes the new `ipfs_cid` in its own metadata (key 674), keyed by `token_name`.

---

## Core types (summary)

**AccumulatorDatum** (on-chain, minimal) — stores only the two fields required for on-chain validation:
- `root_hash` — current tree root (needed for on-chain proof validation by user contracts)
- `policy_id` — `PolicyId` (`ByteArray`, 28 bytes) of the one-shot minting policy that produced the `TreeLifecycleToken`; combined with `token_name` it forms the full `AssetClass`; stored to prevent an attacker from substituting a token with the same asset name under a different policy (INV-4)
- `token_name` — asset name (`ByteArray`, 28 bytes) of the `TreeLifecycleToken` for this tree; combined with `policy_id` uniquely identifies the tree on-chain (INV-4)

> `prev_tx_ref` is absent: chain continuity is guaranteed by the token. `tree_index` is absent: it is encoded inside `token_name`. `descriptor` is absent: fixed at genesis in minting tx metadata. `binding_hash` is absent: IPFS CID guarantees integrity. `ipfs_cid` is absent: published in each spend tx's metadata (key 674), keyed by `token_name`.

**AccumulatorObject** (off-chain, full state) — integer-keyed CBOR map stored permanently in IPFS:
- Key 0: `schema_version` — format version for forward compatibility
- Key 1: `tree_type` — tree implementation identifier
- Key 2: `hash_algorithm` — hash function used
- Key 3: `root_hash` — current root (mirrors datum for verification)
- Key 4: `leaf_count` — current leaf count
- Key 5: `leaves` — complete leaf data (tree-specific serialization)
- Key 6: `internal_nodes` — merkle proof nodes (tree-specific serialization)
- Key 7: `token_name` — asset name (`ByteArray`) of the `TreeLifecycleToken` for this tree (anchors this IPFS object to its on-chain identity; mirrors `datum.token_name`; VERIF-5)

**Transaction Metadata** (Cardano standard key 674):

*Genesis (minting) transaction only:*
- **`descriptor`** (required per tree) — `AccumulatorDescriptor` (tree_type, hash_algorithm, schema_version) for each minted tree; keyed by `token_name`. Fixed for the lifetime of the tree; never repeated in subsequent transactions.
- **`ipfs_cid`** (required per tree) — CID of the initial `AccumulatorObject`; keyed by `token_name`.
- Indexing hints (optional) — labelling which `token_name` corresponds to which domain object.

*Each spend transaction:*
- **`ipfs_cid`** (required per tree updated) — CID of the new `AccumulatorObject`; keyed by `token_name`.
- Note: NOT used for storing full AccumulatorObject (respects 16KB limit); see IPFS for complete state.

**AccumulatorAction** — user contract actions: InitAccumulator, RemoveAccumulator.

---

## TreeLifecycleToken — Minting Policy

Each tree's lifecycle is anchored by a **TreeLifecycleToken**: a native asset minted under a one-shot minting policy that guarantees the token can never be minted again after the genesis transaction.

### One-shot guarantee (the trick)

The minting policy is **parameterized by a seed `OutputReference`** — a specific UTxO that must be spent in the same transaction as the mint. Because a UTxO can only be spent once on Cardano, the policy can only fire once. This gives:

- **Mint exactly 1 token** — for a single-tree `InitAccumulator`: spend the seed UTxO, mint 1 token.
- **Mint exactly X tokens** — for a multi-tree `InitAccumulator` (list of `AccumulatorDatum` values): spend the seed UTxO, validate `mint_quantity == len(declared_datums)`, mint one token per datum, each with a distinct asset name.

The on-chain policy checks:
1. Seed UTxO (`policy_param`) is consumed in this transaction.
2. For each minted asset name, exactly one token is minted (`qty == 1`).
3. Every minted asset name matches `blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ counter_bytes)[0..28]` for a unique counter `i` in `0..N-1`.
4. Each corresponding `AccumulatorDatum` in the output carries a `token_name` equal to the asset name of its paired token.
5. No additional tokens under this policy are minted beyond what is declared.

### Asset name encoding

Each token's **asset name** uniquely identifies a tree for its entire lifetime:

```
asset_name(i) = blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ uint_to_bytes(i))[0..28]
```

`i` is a zero-based counter assigned positionally to each tree in a multi-tree genesis (`0..N-1`). For a single-tree genesis, `i = 0`. The result is a 28-byte prefix of Blake2b-256, fits within Cardano's 32-byte asset name limit, and is collision-resistant.

The `token_name` stored in each `AccumulatorDatum` is exactly this value — it is the sole on-chain identity of the tree.

### Lifecycle role

| Action | Token behaviour |
|---|---|
| `InitAccumulator` | Mint token(s); lock each in the new accumulator UTxO |
| `RemoveAccumulator` | Token must be burned (quantity −1) in the same transaction |

### Relation to existing design

- **Replaces chain traversal for genesis resolution (D-04)**: a verifier no longer needs to follow any chain. The token's `policy_id` encodes the seed `OutputReference`; the `token_name` encodes the per-tree counter. Together they identify the genesis unambiguously in O(1).
- **Replaces INV-3 (chain continuity) and `tree_index` / `prev_tx_ref` in the datum**: the token is the complete tree identity. `datum.token_name` mirrors the token's asset name, tying each datum to its token without any separate `tree_index` or `prev_tx_ref` field.
- **Burn on removal as finality proof**: the absence of the token under a known `policy_id` is machine-verifiable proof that `RemoveAccumulator` has been executed.

---

## On-Chain Validator Invariants (implemented by smart contracts using this library)

These invariants are structural checks enforced on-chain. User contracts call library functions (in `lib/accumulator/invariants.ak`) to validate these properties:

> INV-3 (prev_tx_ref chain continuity) is removed: the `TreeLifecycleToken` can exist at exactly one UTxO at a time; requiring it to be re-locked in the output (INV-8) already enforces a linear, unbranchable progression.

INV-4: `datum.policy_id` and `datum.token_name` must together equal the `AssetClass` of the `TreeLifecycleToken` consumed and re-locked in this transaction — i.e., the UTxO must contain exactly 1 token of `AssetClass(datum.policy_id, datum.token_name)`. Both fields are set at `InitAccumulator` and are immutable for the lifetime of the tree. Validating both prevents a token-name spoofing attack where an attacker mints a token with the same `token_name` under a different `policy_id`.

INV-6: `len(root_hash)` must be a valid hash length (32 or 64 bytes).

> INV-1, INV-2 (descriptor immutability) and INV-5 (binding_hash length) are removed: `descriptor` and `binding_hash` are no longer present in the datum. Descriptor immutability is enforced by the one-shot minting policy (the token can only be minted once; the genesis metadata is immutable). IPFS integrity is guaranteed by content addressing.

INV-7: On `InitAccumulator`, exactly one `TreeLifecycleToken` must be minted per declared datum, under the policy parameterized by the seed `OutputReference` consumed in this transaction. The asset name of each token must equal `blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ uint_to_bytes(i))[0..28]` for counter `i`. Each output `AccumulatorDatum.token_name` must equal the asset name of its paired token.

INV-8: On `RemoveAccumulator`, the `TreeLifecycleToken` matching the tree's genesis identity must be burned (quantity exactly −1) in the same transaction.

## Off-Chain Verification Checks (performed by this library)

VERIF-1: IPFS content integrity — guaranteed by content addressing: the CID is the cryptographic hash of the `AccumulatorObject` content. Fetching by CID and receiving the data is sufficient proof of integrity; no separate on-chain `binding_hash` is required. The `ipfs_cid` is obtained from the spend transaction's metadata (key 674, keyed by `token_name`).

VERIF-2: Proof verification — tree-specific membership/non-membership proof validation via `ProofVerifier` trait, using the `root_hash` from the datum.

VERIF-3: CBOR canonical encoding — all off-chain objects must serialize and deserialize to canonical form.

VERIF-4: Accumulated state reconstruction — replay accumulator transitions to validate historical consistency and state lineage.

VERIF-5: IPFS anchoring consistency — `AccumulatorObject.token_name` must equal `datum.token_name`. This ensures the IPFS object is correctly bound to its on-chain tree identity and cannot be swapped across trees.

VERIF-6: Descriptor resolution — the `AccumulatorDescriptor` (tree_type, hash_algorithm, schema_version) is resolved by looking up the `TreeLifecycleToken`'s minting transaction metadata (key 674, keyed by `token_name`). The resolved descriptor is used to select the correct `ProofVerifier` and hash algorithm for off-chain verification.

---

## Decisions and rationale (memory)

D-01: Generic abstraction for tree types — choose appropriate algorithm per use case (MMR for append-only, Sparse for non-membership).

D-02: IPFS + Filecoin for addressability and durability respectively; `ipfs_cid` published in transaction metadata (key 674), NOT stored in datum (off-chain only).

D-03: ~~Unidirectional binding hash~~ — **removed**. `binding_hash` has been eliminated from the datum. IPFS content integrity is guaranteed by content addressing (the CID is the cryptographic hash of the AccumulatorObject). The AccumulatorObject MUST NOT contain its own CID (circular). The `token_name` in the AccumulatorObject (key 7) anchors it to its on-chain tree identity (VERIF-5).

D-04: Tree identity = `AssetClass(policy_id, token_name)`. The `policy_id` is the hash of the one-shot minting script parameterized by the seed `OutputReference`; the `token_name` is `blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ uint_to_bytes(i))[0..28]`. This pair is globally unique, permanently immutable, and derivable from chain data. Neither `prev_tx_ref` nor `tree_index` are needed as separate datum fields — the `token_name` already encodes genesis identity and per-tree distinction.

D-05: Filecoin deals as primary persistence; optional `filecoin_deal_id` in datums.

D-06: `AccumulatorDatum` uses Plutus Data constructor index `142` via Aiken's `@tag(142)` annotation. This allows user validators to embed the type inside larger datum structures and identify it by constructor index without conflicts.

D-07: `AccumulatorAction` uses Plutus Data constructor index `142` via Aiken's `@tag(142)` annotation, mirroring the datum convention so redeemers can be embedded in the same composable fashion.

D-08: Library scope — this is an off-chain verification library, not a smart contract. It provides reusable types, proof verifiers, and CBOR utilities. Smart contracts using the library implement on-chain invariant checks (INV-4, INV-6, INV-7, INV-8) and authorization logic. Off-chain applications use the library for proof construction, IPFS content validation (via CID), descriptor resolution from genesis metadata, and state transition verification.

D-09: Trait-based tree implementations — each tree type (built-in or custom) implements a `ProofVerifier` interface. Built-in trees included in library; custom trees can be user-provided.

D-10: Built-in trees (StandardMerkle, MMR, SparseMerkle, MerklePatriciaForestry) live in the library repo. Off-chain integration tools (IPFS client, proof constructor, verifier) are optional, added in follow-up phases.

D-11: Transaction metadata strategy — `AccumulatorDatum` contains 3 on-chain fields: `root_hash`, `policy_id`, `token_name`. `descriptor`, `binding_hash`, `prev_tx_ref`, `tree_index`, and `ipfs_cid` are all absent from the datum. `descriptor` is published once in the minting transaction's metadata (key 674, keyed by `token_name`) and never changes. `ipfs_cid` is published in each spend transaction's metadata (key 674, keyed by `token_name`). IPFS integrity is guaranteed by content addressing. Off-chain validation: (1) resolve `descriptor` from the `TreeLifecycleToken`'s minting transaction metadata keyed by `token_name` (VERIF-6), (2) find the tx that produced the UTxO via chain indexing, (3) read `ipfs_cid` from its metadata keyed by `token_name`, (4) fetch AccumulatorObject from IPFS (integrity guaranteed by CID), (5) verify `AccumulatorObject.token_name == datum.token_name` (VERIF-5), (6) apply tree proof validation (VERIF-2) using `root_hash` from datum.

D-12: Multi-tree UTxOs — a single UTxO may carry multiple `AccumulatorDatum` entries as a list (each encoded with constructor index 142). Each datum is distinguished by its `token_name` field, which is the asset name of its paired `TreeLifecycleToken`. The on-chain validator checks that for each datum in the output list, a token with that asset name is present in the UTxO (INV-4). There is no `tree_index` field — the `token_name` is both the per-tree identifier and the positional distinguisher.

D-13: TreeLifecycleToken (one-shot minting policy) — each tree's lifecycle is tracked by a native asset minted under a policy parameterized by a seed `OutputReference`. This provides: (1) scarcity guarantee — the policy can only fire once (seed UTxO can only be spent once); (2) O(1) genesis resolution — the `policy_id` encodes the seed `OutputReference`, the `token_name` is `blake2b_256(seed ++ counter)[0..28]`; (3) complete tree identity — `AssetClass(policy_id, token_name)` replaces `prev_tx_ref + tree_index` entirely; (4) lifecycle finality — burning the token is machine-verifiable proof of `RemoveAccumulator`. For multi-tree genesis, the policy mints exactly `N` tokens (one per datum), each with a distinct counter-derived asset name. INV-7 and INV-8 enforce this on-chain.

---

## Library Architecture: On-Chain & Off-Chain Components

### Data Storage Strategy: Cardano-Optimized Separation

**Principle:** Minimize on-chain data (expensive UTxO storage) while preserving complete verifiability off-chain.

| Data | Location | Size | On-Chain Cost | Rationale |
|------|----------|------|---------------|-----------|
| `root_hash` | UTxO datum | 32-64 bytes | High (essential) | Required for on-chain proof validation by user contracts |
| `policy_id` | UTxO datum | 28 bytes | High (essential) | `PolicyId` of the one-shot minting policy; prevents token-name spoofing; validated by INV-4 |
| `token_name` | UTxO datum | 28 bytes | High (essential) | Asset name of `TreeLifecycleToken`; together with `policy_id` forms the unique tree `AssetClass`; INV-4 |
| **Total Datum Size** | **UTxO** | **~88 bytes** | **Minimal** | Only validation-critical fields |
| `descriptor` | Minting tx metadata (674) | ~10 bytes | One-time / free | Published once at genesis; fixed for lifetime; keyed by `token_name` |
| `ipfs_cid` (genesis) | Minting tx metadata (674) | ~36 bytes | One-time / free | Initial AccumulatorObject CID; keyed by `token_name` |
| `ipfs_cid` (per spend) | Spend tx metadata (674) | ~36 bytes | Low | New AccumulatorObject CID per state transition; keyed by `token_name` |
| Full `AccumulatorObject` | IPFS | ~1-100 KB | Free (off-chain) | Complete tree state + `token_name` anchor (key 7); integrity guaranteed by CID; permanent storage via Filecoin |

### On-Chain Components (for Smart Contract Developers)

The library provides **on-chain invariant checking**:
1. **Types** — `AccumulatorDatum` (3 fields: `root_hash`, `policy_id`, `token_name`), `AccumulatorAction`, encoded as Plutus Data with constructor index `142` (via `@tag(142)`). `AccumulatorDescriptor` is an off-chain-only type resolved from genesis metadata.
2. **Invariant validators** (in `lib/accumulator/invariants.ak`) — functions to check INV-4 (`AssetClass(datum.policy_id, datum.token_name)` matches token in UTxO), INV-6 (root_hash length), INV-7 (minting policy), INV-8 (burn on removal).
3. **Type utilities** — helper functions for type coercion, datum extraction, and action validation.

**User Contracts Implement**:
- Authorization logic (who may update the accumulator).
- On-chain integration with other protocols or state machines.

### Off-Chain Components (for Verification & Proof Construction)

The library provides **off-chain verification utilities**:
1. **CBOR serialization** — canonical encode/decode of `AccumulatorObject` and related structures (per Cardano standards).
2. **Proof verifiers** (trait-based) — tree-specific membership/non-membership proof validation using `root_hash` from the datum.
3. **Descriptor resolution** — look up the `TreeLifecycleToken`'s minting transaction; read `AccumulatorDescriptor` from its metadata (key 674, keyed by `token_name`) to recover `tree_type`, `hash_algorithm`, and `schema_version` (VERIF-6).
4. **IPFS integration** — resolve `ipfs_cid` from each spend transaction's metadata (key 674, keyed by `token_name`); fetch `AccumulatorObject` from IPFS (content integrity guaranteed by CID — VERIF-1); verify `AccumulatorObject.token_name == datum.token_name` (VERIF-5).
5. **State reconstruction** — reconstruct accumulator state by: (a) resolving the descriptor from the minting tx keyed by `token_name` (VERIF-6), (b) finding the tx that produced the UTxO via chain indexing, (c) reading `ipfs_cid` from its metadata keyed by `token_name`, (d) fetching `AccumulatorObject` from IPFS, (e) verifying `AccumulatorObject.token_name` (VERIF-5), (f) replaying subsequent on-chain transactions (redeemer + new datum), (g) validating each transition including INV-4.
6. **Transaction metadata handling** — parse Cardano transaction metadata (key 674) for `ipfs_cid` (per spend, keyed by `token_name`) and `descriptor` + initial `ipfs_cid` (genesis/minting tx, keyed by `token_name`).

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
  let accumulator: AccumulatorDatum = datum     // constructor index 142 via @tag(142)
  let action: AccumulatorAction = redeemer      // constructor index 142 via @tag(142)
  
  // On-chain: check datum token_name matches the token in the UTxO + root_hash length
  invariants.validate_all(accumulator, ctx)
  && my_authorization_logic(ctx)
}
```

**Off-Chain (Proof Verification Example)**:
```typescript
// Resolve descriptor from genesis (minting) transaction metadata keyed by token_name
const descriptor = await resolveDescriptor(tokenPolicyId, tokenName);

// Resolve ipfs_cid from the spend transaction's metadata keyed by token_name
const ipfsCid = await resolveIpfsCid(spendTxHash, tokenName);

// Fetch from IPFS — CID guarantees content integrity
const accumulatorObject = await ipfs.get(ipfsCid);

// Off-chain verification
const isValid =
  accumulatorObject.token_name === datum.token_name &&           // VERIF-5
  StandardMerkle.verifyMembership(datum.root_hash, leaf, proof); // VERIF-2
```

---

## Next recommended implementation steps

### Phase 1: Core Library (On-Chain & Off-Chain Shared)

1. Initialize Aiken project and `aiken.toml`.
2. Implement `lib/accumulator/types.ak` (all core types).
3. Implement `lib/accumulator/cbor.ak` for canonical CBOR serialization/deserialization.
4. Implement `lib/accumulator/invariants.ak` with INV-4, INV-6, INV-7, INV-8 validators and unit tests.

### Phase 2: Built-in Tree Implementations (Off-Chain Verification)

5. Implement `lib/trees/standard_merkle.ak` (reference implementation, membership proof).
6. Implement `lib/trees/merkle_mmr.ak` (append-only accumulator).
7. Implement `lib/trees/sparse_merkle.ak` (membership + non-membership proofs).
8. Implement `lib/trees/merkle_patricia_forestry.ak` (complex tree structures).

### Phase 3: Documentation & Examples (On-Chain Integration)

9. Create example validator in `examples/example_validator.ak` (demonstrates on-chain invariant checking + authorization).
10. Document descriptor resolution from genesis metadata and IPFS CID-based integrity model.
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
- Q-06: Transaction metadata schema (key 674) — define the exact CBOR structure for: (a) the genesis/minting tx entry containing `descriptor` and initial `ipfs_cid` per `token_name`; (b) spend tx entries containing `ipfs_cid` per `token_name`. The key is the `token_name` bytes.
- Q-07: State reconstruction performance — for chains with many accumulator update transactions, should off-chain indexing cache intermediate AccumulatorObject snapshots at checkpoints, or always reconstruct from the nearest IPFS snapshot + transaction replay?
- Q-08: ~~`tree_index` assignment order~~ — **superseded**. `tree_index` has been removed from the datum. The positional counter `i` used during minting to derive each `token_name` is an implementation detail of the minting policy; it is not stored anywhere and does not need external convention.

- Q-09: Seed UTxO selection for the one-shot minting policy — should the off-chain builder be free to choose any wallet UTxO as the seed, or should the protocol mandate a specific selection convention (e.g., the first input by lexicographic `OutputReference` order)?

- Q-10: Token custody — should the `TreeLifecycleToken` be co-located with the accumulator UTxO (same address, same datum) or can it be held separately (e.g., in a dedicated token UTxO at the same validator)? Co-location is simpler; separate custody could allow token-gated access patterns for external contracts.

---

## References

This file is the canonical reference for agents and decisions; previous long-form docs were consolidated here.
