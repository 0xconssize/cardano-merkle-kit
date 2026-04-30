# DESIGN.md — Cardano Accumulator SDK

---

## Project description

An off-chain verification library that provides core primitives for accumulators on Cardano. The library is composed of two complementary parts:

**On-chain Validation** (Smart Contract Component) — three distinct modules under `lib/accumulator/`:

- **Shared types** (`types.ak`) — `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorRedeemer`, and `AccumulatorOperation`; tagged 142 for safe embedding inside user-defined datum and redeemer structures via Plutus Data constructor matching.
- **Minting policy** (`lifecycle_policy.ak`) — one-shot minting policy for the `AccumulatorLifecycleToken`. Handles `InitAccumulator` (genesis mint) and the remint side of `CheckpointAccumulator`; enforces seed UTxO consumption and asset name derivation (INV-3, INV-6 mint-side).
- **Invariant library** (`invariants.ak`) — reusable functions called by user spending validators. Enforces structural invariants for `UpdateAccumulator`, `CheckpointAccumulator` (burn-and-transition side), and `RemoveAccumulator`. Does NOT verify IPFS content or cryptographic proofs.
- Chain continuity is guaranteed by the one-shot `AccumulatorLifecycleToken`; user contracts compose invariant checks with their own authorization logic.

**Off-chain Monitoring**:
- Provides types, implementations, and utilities for accumulator handling
- Monitors accumulator state transitions and detects unauthorized data manipulation
- Resolves the `AccumulatorDescriptor` from the genesis (minting) transaction metadata
- Validates IPFS content integrity via CID
- Emits structured state change events and computes element diffs between transitions
- Abstracts over accumulator types via a pluggable `AccumulatorValidator` interface

The library is **pure and composable** (not a smart contract). It provides:
- **Types**: `AccumulatorDescriptor`, `AccumulatorDatum`, `AccumulatorAction`, `AccumulatorObject`
- **Built-in accumulator implementations**: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry (with trait extensibility)
- **CBOR serialization**: canonical encoding for off-chain accumulator objects
- **Observability**: state change detection, structured event emission, and element diff computation

> `AccumulatorDescriptor` is resolved off-chain from the genesis (minting) transaction metadata; it is not stored in the on-chain datum.

User validators import the library's on-chain components (invariant checks) and combine them with custom authorization logic. Off-chain applications use the library's monitoring components to observe state transitions, detect mutations, and reconstruct accumulator history.

---

## Stack

- Blockchain: Cardano (mainnet/preprod)
- Smart contracts: Aiken (compiles to UPLC / Plutus Core)
- Plutus target: Plutus V3
- Off-chain storage: IPFS (CIDv1) + Filecoin
- Serialization: CBOR canonical encoding
- Off-chain tooling: TypeScript (EffectTs)
- Testing: Aiken tests + property-based tests where applicable

---

## Non-negotiable design principles

1. Clear separation of on-chain validation vs off-chain verification

   **On-chain (smart contract):** Validates that `datum.policy_id` and `datum.token_name` together match the `AssetClass` of the `AccumulatorLifecycleToken` present in the UTxO (INV-1), `state_digest` bounds (INV-2), and token presence/burn (INV-3, INV-4). Storing both fields prevents an attacker from substituting a token with the same asset name minted under a different policy. Chain continuity is guaranteed by the one-shot token.
   **Off-chain:** Handles IPFS content integrity via CID, accumulator state change monitoring, state reconstruction, and accumulator identity anchoring. Resolves the `AccumulatorDescriptor` by looking up the genesis (minting) transaction's metadata.
   Each `AccumulatorObject` stored in IPFS MUST embed the `token_name` of its `AccumulatorLifecycleToken` so that off-chain verifiers can identify which accumulator it belongs to (VERIF-5). The IPFS CID itself MUST NOT appear in the object (circular reference).

2. Abstraction of accumulator type and type-specific configuration

   The protocol treats `accumulator_type` as the primary discriminant in the `AccumulatorDescriptor`. Type-specific configuration is carried in an open `params` map whose keys are defined per accumulator type — accumulators requiring no configuration publish an empty map. Supported initial types: StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry. For Merkle-based variants, `params` contains `hash_algorithm`; supported algorithms: Blake2b256, Blake2b224, SHA2_256, SHA3_256, Keccak256, Poseidon (Poseidon verification is off-chain). Other accumulator families (e.g. RSA, KZG) declare their own `params` keys (e.g. `modulus_bits`, `elliptic_curve`).

   The `AccumulatorDescriptor` is published once in the genesis (minting) transaction's metadata (key 674) and is immutable for the lifetime of the accumulator. It is resolved off-chain by looking up the `AccumulatorLifecycleToken`'s minting transaction.

3. Composability of Datum and Redeemer via Plutus Data constructor tag `142`

   Tag `142` is the Plutus Data constructor index assigned to `AccumulatorDatum` and `AccumulatorAction` via Aiken's `@tag(142)` annotation. Plutus Data is a recursive sum type; Aiken allows traversing it and pattern-matching on constructor indices at runtime, making it possible to locate and decode a known library type by its tag even when it is embedded inside an arbitrary outer structure. Constructor index 142 is reserved exclusively for the library's types, ensuring no conflict with user-defined constructors.

   This enables two standard composition patterns used throughout the library:

   - **Datum embedding**: a user validator's datum type includes `AccumulatorDatum` as a field; the spending validator pattern-matches on constructor 142 to extract it and passes it to `invariants.ak`.
   - **Redeemer embedding**: a user validator's redeemer type includes `AccumulatorAction` as a field; the spending validator pattern-matches on constructor 142 to extract it and dispatch on the variant. The `UpdateAccumulator` variant carries `List<AccumulatorOperation>` directly, so all operation data is self-contained in the action value itself — no additional wrapper type is needed.

   User contracts are not required to use `AccumulatorDatum` or `AccumulatorAction` as top-level types — they embed them inside their own structures and extract them by tag at runtime.

4. Library provides reusable components, not on-chain validators

   This library provides reusable components (types, CBOR serialization, and state monitoring utilities) that smart contract developers and off-chain integrators use. User contracts are responsible for authorization logic and for composing on-chain invariant checks with authorization.

5. Cardano storage optimization: minimal datum, IPFS for full state, genesis metadata for descriptor

   Cardano penalizes on-chain storage (UTxO size affects transaction costs; metadata has a 16KB limit). `AccumulatorDatum` stores only the three fields required for on-chain validation: `state_digest`, `policy_id`, and `token_name`. The `descriptor` is published once in the minting transaction's metadata (key 674) and is fixed for the lifetime of the accumulator. The full accumulator state is stored in IPFS (integrity guaranteed by content addressing). Each spend transaction publishes the new `ipfs_cid` in its own metadata (key 674), keyed by `token_name`.

---

## Core types (summary)

**AccumulatorDatum** (on-chain, minimal) — three fields required for on-chain validation:
- `state_digest` — cryptographic commitment to the current accumulator state; enables off-chain monitors to detect unauthorized state changes (VERIF-2)
- `policy_id` — `PolicyId` (`ByteArray`, 28 bytes) of the one-shot minting policy that produced the `AccumulatorLifecycleToken`; combined with `token_name` forms the full `AssetClass` (INV-1)
- `token_name` — asset name (`ByteArray`, 28 bytes) of the `AccumulatorLifecycleToken`; combined with `policy_id` uniquely identifies the accumulator on-chain (INV-1)

**AccumulatorObject** (off-chain, full state) — integer-keyed CBOR map stored permanently in IPFS:
- Key 0: `schema_version` — format version for forward compatibility
- Key 1: `accumulator_type` — accumulator implementation identifier
- Key 2: `params` — type-specific configuration (mirrors `AccumulatorDescriptor.params`; e.g. `{"hash_algorithm": "blake2b256"}` for Merkle variants; empty map `{}` for types with no configuration)
- Key 3: `state_digest` — current accumulator state commitment (mirrors `datum.state_digest`; used for off-chain consistency checks)
- Key 4: `element_count` — current element count
- Key 5: `elements` — complete element data (accumulator-specific serialization)
- Key 6: `token_name` — asset name (`ByteArray`) of the `AccumulatorLifecycleToken` for this accumulator (anchors this IPFS object to its on-chain identity; mirrors `datum.token_name`; VERIF-5)

**Transaction Metadata** (Cardano standard key 674):

*Genesis (minting) transaction only:*
- **`descriptor`** (required per accumulator) — `AccumulatorDescriptor` (accumulator_type, params) for each minted accumulator; keyed by `token_name`. `params` is an open CBOR map with type-specific keys (e.g. `hash_algorithm` for Merkle variants; empty for types with no configuration). Fixed for the lifetime of the accumulator; never repeated in subsequent transactions.
- **`ipfs_cid`** (required per accumulator) — CID of the initial `AccumulatorObject`; keyed by `token_name`.
- Indexing hints (optional) — labelling which `token_name` corresponds to which domain object.

*Each `CheckpointAccumulator` transaction:*
- **`ipfs_cid`** (required per checkpointed accumulator) — CID of the full `AccumulatorObject` snapshot at the checkpoint; keyed by the **new** `token_name`. This is the authoritative IPFS anchor for the new epoch.
- Note: The checkpoint transaction burns the old `AccumulatorLifecycleToken` and mints a new one; both appear in the same transaction's mint/burn events and establish the epoch boundary on-chain (VERIF-7).

**AccumulatorAction** (`@tag(142)`) — the four lifecycle actions, each applicable to a specific on-chain contract scope:

```aiken
@tag(142)
type AccumulatorAction {
  InitAccumulator
  UpdateAccumulator { operations: List<AccumulatorOperation> }
  CheckpointAccumulator
  RemoveAccumulator
}
```

| Action | Minting policy | Spending validator | Description |
|---|:---:|:---:|---|
| `InitAccumulator` | yes | no | Genesis: consume seed UTxO, mint lifecycle token(s), lock initial datum(s) |
| `UpdateAccumulator` | no | yes | Spend UTxO, update `state_digest`, re-lock token; carries the list of element operations inline |
| `CheckpointAccumulator` | yes (remint side) | yes (burn-and-transition side) | Two-script action in one transaction: spending validator authorises the token burn and state transition; minting policy authorises the replacement token mint |
| `RemoveAccumulator` | no | yes | Spend the UTxO holding the token; the spending validator authorises burning it (quantity −1) in the same transaction |

`RemoveAccumulator` is a spending validator action: on Cardano, burning a token requires spending the UTxO that holds it; the spending validator confirms the burn (INV-4).

`CheckpointAccumulator` is the only action that must satisfy two scripts simultaneously in the same transaction: the spending validator handles the burn side (old token, old datum), and the minting policy handles the mint side (new token, new datum).

**AccumulatorOperation** (on-chain, embedded in `UpdateAccumulator`) — a single element mutation with its accumulator-specific proof:
- `element: Data` — the element being added or removed; opaque Plutus Data, accumulator-type-specific; not interpreted on-chain
- `proof: Data` — membership or non-membership proof for the element; opaque Plutus Data, accumulator-type-specific (e.g. Merkle path encoded as a `List<ByteArray>`, RSA witness or KZG opening proof encoded as bytes wrapped in `Data`); not interpreted on-chain
- `op: OpType` — `AddElement` or `RemoveElement`

The on-chain validator records these fields in the redeemer without inspecting `element` or `proof`. Off-chain verifiers dispatch on `accumulator_type` (from the resolved `AccumulatorDescriptor`) to decode and verify the proof.

---

## AccumulatorLifecycleToken — Minting Policy

Each accumulator's lifecycle is anchored by an **AccumulatorLifecycleToken**: a native asset minted under a one-shot minting policy that guarantees the token can never be minted again after the genesis transaction.

### One-shot guarantee

The minting policy is **parameterized by a seed `OutputReference`** — a specific UTxO that must be spent in the same transaction as the mint. Because a UTxO can only be spent once on Cardano, the policy can only fire once. This gives:

- **Mint exactly 1 token** — for a single-accumulator `InitAccumulator`: spend the seed UTxO, mint 1 token.
- **Mint exactly X tokens** — for a multi-accumulator `InitAccumulator` (list of `AccumulatorDatum` values): spend the seed UTxO, validate `mint_quantity == len(declared_datums)`, mint one token per datum, each with a distinct asset name.

The on-chain policy checks:
1. Seed UTxO (`policy_param`) is consumed in this transaction.
2. For each minted asset name, exactly one token is minted (`qty == 1`).
3. Every minted asset name matches `blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ counter_bytes)[0..28]` for a unique counter `i` in `0..N-1`.
4. Each corresponding `AccumulatorDatum` in the output carries a `token_name` equal to the asset name of its paired token.
5. No additional tokens under this policy are minted beyond what is declared.

### Asset name encoding

Each token's **asset name** uniquely identifies an accumulator for its entire lifetime:

```
asset_name(i) = blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ uint_to_bytes(i))[0..28]
```

`i` is a zero-based counter assigned positionally to each accumulator in a multi-accumulator genesis (`0..N-1`). For a single-accumulator genesis, `i = 0`. The result is a 28-byte prefix of Blake2b-256, fits within Cardano's 32-byte asset name limit, and is collision-resistant.

The `token_name` stored in each `AccumulatorDatum` is exactly this value — it is the sole on-chain identity of the accumulator.

### Lifecycle role

| Action | Token behaviour |
|---|---|
| `InitAccumulator` | Mint token(s) under new one-shot policy; lock each in the new accumulator UTxO |
| `UpdateAccumulator` | Token stays locked; `List<AccumulatorOperation>` (element + proof per operation) recorded in redeemer |
| `CheckpointAccumulator` | Burn old token (−1); remint replacement token under new one-shot policy (new seed UTxO); new datum carries new `policy_id` + `token_name`; full `AccumulatorObject` published to IPFS |
| `RemoveAccumulator` | Token must be burned (quantity −1) in the same transaction |

---

## On-Chain Validator Invariants (implemented by smart contracts using this library)

These invariants are structural checks enforced on-chain. User contracts call library functions (in `lib/accumulator/invariants.ak`) to validate these properties:

INV-1: `datum.policy_id` and `datum.token_name` must together equal the `AssetClass` of the `AccumulatorLifecycleToken` consumed and re-locked in this transaction — i.e., the UTxO must contain exactly 1 token of `AssetClass(datum.policy_id, datum.token_name)`. Both fields are set at `InitAccumulator` and are immutable for the lifetime of the accumulator. Validating both prevents a token-name spoofing attack where an attacker mints a token with the same `token_name` under a different `policy_id`.

INV-2: `len(state_digest)` must be between 1 and 512 bytes. The exact expected byte length for a given accumulator type is declared in `AccumulatorDescriptor.params` and validated off-chain.

INV-3: On `InitAccumulator`, exactly one `AccumulatorLifecycleToken` must be minted per declared datum, under the policy parameterized by the seed `OutputReference` consumed in this transaction. The asset name of each token must equal `blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ uint_to_bytes(i))[0..28]` for counter `i`. Each output `AccumulatorDatum.token_name` must equal the asset name of its paired token.

INV-4: On `RemoveAccumulator`, the `AccumulatorLifecycleToken` matching the accumulator's genesis identity must be burned (quantity exactly −1) in the same transaction.

INV-5: On `UpdateAccumulator`, the redeemer must contain a non-empty `List<AccumulatorOperation>`. Each operation carries `element: Data`, `proof: Data`, and `op: OpType`. The on-chain validator does not interpret `element` or `proof` (cryptographic proof verification is off-chain); it checks that (a) the list is non-empty, (b) `state_digest` bounds are preserved (INV-2), and (c) the `AccumulatorLifecycleToken` is re-locked in the output UTxO.

INV-6: On `CheckpointAccumulator`, the transaction must (a) burn exactly 1 token of `AssetClass(old_datum.policy_id, old_datum.token_name)`, (b) mint exactly 1 replacement token under a new one-shot policy parameterized by a new seed `OutputReference` consumed in this transaction — the new `token_name` is `blake2b_256(new_seed_tx_hash ++ new_seed_output_index_bytes ++ 0x00)[0..28]`, (c) produce an output datum with `policy_id` and `token_name` equal to the new token's `AssetClass`, and (d) satisfy INV-2 on the new `state_digest`.

## Off-Chain Verification Checks (performed by this library)

VERIF-1: IPFS content integrity — guaranteed by content addressing: the CID is the cryptographic hash of the `AccumulatorObject` content. Fetching by CID and receiving the data is sufficient proof of integrity. The `ipfs_cid` is obtained from the spend transaction's metadata (key 674, keyed by `token_name`).

VERIF-2: State change validation — the `state_digest` in the new datum must correspond to a structurally sound transition from the previous `AccumulatorObject`. The off-chain validator fetches both objects from IPFS and confirms that the element set is internally consistent with `state_digest` for the declared `accumulator_type`.

VERIF-3: CBOR canonical encoding — all off-chain objects must serialize and deserialize to canonical form.

VERIF-4: State reconstruction — replay accumulator transitions to validate historical consistency and state lineage.

VERIF-5: IPFS anchoring consistency — `AccumulatorObject.token_name` must equal `datum.token_name`. This ensures the IPFS object is correctly bound to its on-chain accumulator identity and cannot be swapped across accumulators.

VERIF-6: Descriptor resolution — the `AccumulatorDescriptor` (accumulator_type, params) is resolved by looking up the `AccumulatorLifecycleToken`'s minting transaction metadata (key 674, keyed by `token_name`). The resolved descriptor is used to extract type-specific configuration from `params` (e.g. `hash_algorithm` for Merkle variants) for off-chain state change validation.

VERIF-7: Checkpoint lineage traversal — to reconstruct the full history of an accumulator across checkpoint epochs, the off-chain monitor inspects the mint/burn events of each `CheckpointAccumulator` transaction: the burned `token_name` identifies the closing epoch's identity, and the minted `token_name` (under the new `policy_id`) identifies the opening of the next epoch. No metadata or datum field is required; the lineage is fully derivable from chain data. Between two checkpoints, state is reconstructed by starting from the epoch's authoritative IPFS snapshot (keyed by the epoch's `token_name` in the checkpoint tx metadata) and replaying each subsequent `UpdateAccumulator` redeemer's `AccumulatorOperation` list in block order.

---

## Decisions and rationale

D-01: Generic abstraction for accumulator types — choose the appropriate implementation per use case (MerkleMMR for append-only event logs, SparseMerkle for key-value state monitoring).

D-02: IPFS + Filecoin for off-chain storage — IPFS for content-addressed retrieval, Filecoin for persistence. `ipfs_cid` is published in transaction metadata (key 674), not stored in the datum.

D-03: Accumulator identity = `AssetClass(policy_id, token_name)`. The `policy_id` is the hash of the one-shot minting script parameterized by the seed `OutputReference`; the `token_name` is `blake2b_256(seed_tx_hash ++ seed_output_index_bytes ++ uint_to_bytes(i))[0..28]`. This pair is globally unique, permanently immutable, and derivable from chain data. The `token_name` encodes genesis identity and per-accumulator distinction without requiring additional datum fields.

D-04: Filecoin deals as primary persistence; optional `filecoin_deal_id` in datums.

D-05: `AccumulatorDatum` uses Plutus Data constructor index `142` via Aiken's `@tag(142)` annotation. This allows user validators to embed the type inside larger datum structures and identify it by constructor index without conflicts.

D-06: `AccumulatorAction` uses Plutus Data constructor index `142` via Aiken's `@tag(142)` annotation, mirroring the datum convention so redeemers can be embedded in the same composable fashion.

D-07: Library scope — this SDK delivers two modules: an Aiken on-chain library (invariant checking, types) and a TypeScript off-chain package (observability, state monitoring, IPFS integration). Smart contracts using the library implement on-chain invariant checks (INV-1, INV-2, INV-3, INV-4) and authorization logic. Off-chain applications use the TypeScript package for accumulator state monitoring, event-driven change detection, IPFS content validation (via CID), descriptor resolution, and state reconstruction.

D-08: Pluggable accumulator validators — each accumulator type (built-in or custom) implements an `AccumulatorValidator` interface responsible for state transition consistency checks and element diff computation. Built-in validators are included in the TypeScript package; custom implementations can be user-provided.

D-09: Built-in `AccumulatorValidator` implementations (StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry) live in the TypeScript package. Additional off-chain integration tools are added in follow-up phases.

D-10: Transaction metadata strategy — `AccumulatorDatum` contains 3 on-chain fields: `state_digest`, `policy_id`, `token_name`. The `descriptor` is published once in the minting transaction's metadata (key 674, keyed by `token_name`) and is immutable. For `UpdateAccumulator` spend transactions, `ipfs_cid` is optional in metadata (keyed by `token_name`); if absent, state is reconstructable from the redeemer chain. For `CheckpointAccumulator` transactions, `ipfs_cid` is required in metadata (keyed by the **new** `token_name`), providing the authoritative IPFS snapshot for the new epoch. IPFS integrity is guaranteed by content addressing. Off-chain validation: (1) resolve `descriptor` from the `AccumulatorLifecycleToken`'s minting transaction metadata keyed by `token_name` (VERIF-6), (2) identify the nearest preceding checkpoint (or genesis) IPFS snapshot, (3) replay each subsequent `UpdateAccumulator` redeemer's `AccumulatorOperation` list in block order (VERIF-4, VERIF-7), (4) if `ipfs_cid` is present in the tx metadata, fetch `AccumulatorObject` from IPFS and validate anchoring (VERIF-5) and structural consistency (VERIF-2).

D-11: Multi-accumulator UTxOs — a single UTxO may carry multiple `AccumulatorDatum` entries as a list (each encoded with constructor index 142). Each datum is distinguished by its `token_name` field, which is the asset name of its paired `AccumulatorLifecycleToken`. The on-chain validator checks that for each datum in the output list, a token with that asset name is present in the UTxO (INV-1).

D-12: AccumulatorLifecycleToken (one-shot minting policy) — each accumulator's lifecycle is tracked by a native asset minted under a policy parameterized by a seed `OutputReference`. This provides: (1) scarcity guarantee — the policy can only fire once (seed UTxO can only be spent once); (2) O(1) genesis resolution — the `policy_id` encodes the seed `OutputReference`, the `token_name` is `blake2b_256(seed ++ counter)[0..28]`; (3) complete accumulator identity — `AssetClass(policy_id, token_name)` is the sole chain-verifiable identifier; (4) lifecycle finality — burning the token is machine-verifiable proof of `RemoveAccumulator`. For multi-accumulator genesis, the policy mints exactly `N` tokens (one per datum), each with a distinct counter-derived asset name. INV-3 and INV-4 enforce this on-chain.

D-13: `AccumulatorDescriptor` uses an open `params` CBOR map for type-specific configuration. `accumulator_type` is the sole mandatory field; `params` keys are defined per accumulator type and documented in the library. An empty map is valid for types that need no configuration. This allows any accumulator family (Merkle, RSA, KZG, etc.) to declare its own configuration without a schema change. Readers dispatch on `accumulator_type` to interpret `params`. The `AccumulatorObject` mirrors this with key 2 (`params`) for self-description.

D-14: `UpdateAccumulator` carries a `List<AccumulatorOperation>` in the redeemer. Each `AccumulatorOperation` has `element: Data`, `proof: Data`, and `op: OpType`. Both `element` and `proof` are typed as Plutus `Data` — completely opaque to the on-chain validator and never inspected or evaluated on-chain. Off-chain verifiers dispatch on `accumulator_type` (from the resolved `AccumulatorDescriptor`) to decode and verify the proof (e.g., a Merkle path encoded as a `List<ByteArray>` for Merkle variants, or a KZG opening proof encoded as raw bytes wrapped in `Data`). This preserves non-negotiable principle 1 (cryptographic proof verification stays off-chain) while permanently recording the full transition evidence in the redeemer.

D-15: `CheckpointAccumulator` advances the accumulator to a new epoch by burning the existing `AccumulatorLifecycleToken` and reminting a replacement under a new one-shot minting policy parameterized by a fresh seed `OutputReference`. The checkpoint transaction's mint/burn events are the authoritative lineage record: the burned `AssetClass` identifies the closing epoch; the minted `AssetClass` identifies the opening epoch. Off-chain monitors traverse epoch boundaries by following these events on-chain (VERIF-7); no datum or metadata field is required for lineage. The checkpoint transaction must publish a full `AccumulatorObject` IPFS snapshot (keyed by the new `token_name`) so that reconstruction of the new epoch begins from a known state rather than replaying the entire history.

---

## Library Architecture: On-Chain & Off-Chain Components

### Data Storage Strategy: Cardano-Optimized Separation

**Principle:** Minimize on-chain data (expensive UTxO storage) while preserving complete verifiability off-chain.

| Data | Location | Size | On-Chain Cost | Rationale |
|------|----------|------|---------------|-----------|
| `state_digest` | UTxO datum | 1-512 bytes | High (essential) | Cryptographic commitment to accumulator state; enables off-chain monitors to detect unauthorized changes |
| `policy_id` | UTxO datum | 28 bytes | High (essential) | `PolicyId` of the one-shot minting policy; prevents token-name spoofing; validated by INV-1 |
| `token_name` | UTxO datum | 28 bytes | High (essential) | Asset name of `AccumulatorLifecycleToken`; together with `policy_id` forms the unique accumulator `AssetClass`; INV-1 |
| **Total Datum Size** | **UTxO** | **~88 bytes** | **Minimal** | Only validation-critical fields |
| `descriptor` | Minting tx metadata (674) | ~10–50 bytes | One-time / free | Published once at genesis; fixed for lifetime; keyed by `token_name`; size depends on `params` content |
| `ipfs_cid` (genesis) | Minting tx metadata (674) | ~36 bytes | One-time / free | Initial AccumulatorObject CID; keyed by `token_name` |
| `ipfs_cid` (UpdateAccumulator) | Spend tx metadata (674) | ~36 bytes | Optional / Low | Optional AccumulatorObject CID per update; keyed by `token_name`; omitted if operator defers snapshot |
| `AccumulatorOperation` list | Spend tx redeemer | variable | Medium (exec budget) | Element + opaque proof (`Data`) per operation; permanently recorded; enables off-chain state reconstruction without IPFS between checkpoints |
| `ipfs_cid` (CheckpointAccumulator) | Checkpoint tx metadata (674) | ~36 bytes | Low | Required IPFS snapshot for new epoch; keyed by **new** `token_name`; epoch reconstruction starts here |
| Full `AccumulatorObject` | IPFS | ~1-100 KB | Free (off-chain) | Complete accumulator state + `token_name` anchor (key 6); integrity guaranteed by CID; permanent storage via Filecoin |

### On-Chain Components (for Smart Contract Developers)

The library provides **on-chain invariant checking**:
1. **Types** — `AccumulatorDatum` (3 fields: `state_digest`, `policy_id`, `token_name`), `AccumulatorAction`, encoded as Plutus Data with constructor index `142` (via `@tag(142)`). `AccumulatorDescriptor` is an off-chain-only type resolved from genesis metadata.
2. **Invariant validators** (in `lib/accumulator/invariants.ak`) — functions to check INV-1 (`AssetClass(datum.policy_id, datum.token_name)` matches token in UTxO), INV-2 (state_digest bounds), INV-3 (minting policy), INV-4 (burn on removal), INV-5 (update operations list non-empty + token re-lock), INV-6 (checkpoint burn + remint under new seed).
3. **Type utilities** — helper functions for type coercion, datum extraction, and action validation.

**User Contracts Implement**:
- Authorization logic (who may update the accumulator).
- On-chain integration with other protocols or state machines.

### Off-Chain Components (for Monitoring & Observability)

The TypeScript package provides **off-chain monitoring utilities**:
1. **CBOR serialization** — canonical encode/decode of `AccumulatorObject` and related structures (per Cardano standards).
2. **Accumulator state monitoring** — observe UTxO changes for a given `AccumulatorLifecycleToken`; fetch the new `AccumulatorObject` from IPFS on each state transition; validate structural consistency (VERIF-2) and anchoring (VERIF-5).
3. **Descriptor resolution** — look up the `AccumulatorLifecycleToken`'s minting transaction; read `AccumulatorDescriptor` from its metadata (key 674, keyed by `token_name`) to recover `accumulator_type` and `params` (VERIF-6).
4. **IPFS integration** — resolve `ipfs_cid` from each spend transaction's metadata (key 674, keyed by `token_name`); fetch `AccumulatorObject` from IPFS (content integrity guaranteed by CID — VERIF-1).
5. **State reconstruction** — reconstruct accumulator history by: (a) resolving the descriptor from the minting tx (VERIF-6), (b) traversing checkpoint epoch boundaries via mint/burn events (VERIF-7), (c) for each epoch, starting from the epoch's authoritative IPFS snapshot and replaying subsequent `UpdateAccumulator` redeemer operations in block order, (d) validating anchoring (VERIF-5) and structural consistency (VERIF-2) at each IPFS-published snapshot (VERIF-4).
6. **Transaction metadata handling** — parse Cardano transaction metadata (key 674) for `ipfs_cid` (per spend, keyed by `token_name`) and `descriptor` + initial `ipfs_cid` (genesis/minting tx, keyed by `token_name`).
7. **Event emission** — emit structured `AccumulatorChangeEvent` records on each detected state transition, including `txHash`, `previousCid`, `newCid`, and a `stateDiff` describing elements added or removed.

**Built-in AccumulatorValidator implementations** (TypeScript):
- `StandardMerkleValidator` — validates StandardMerkle state transitions and computes element diffs.
- `MerkleMMRValidator` — validates MerkleMMR append-only transitions.
- `SparseMerkleValidator` — validates SparseMerkle key-value state transitions.
- `MerklePatriciaForestryValidator` — validates MerklePatriciaForestry state transitions.

Each implements the `AccumulatorValidator` interface:
```typescript
interface AccumulatorValidator {
  validateTransition(prev: AccumulatorObject, next: AccumulatorObject, params: Params): boolean;
  computeDiff(prev: AccumulatorObject, next: AccumulatorObject): StateDiff;
}
```

**Custom Implementations**: Users can extend the library by implementing `AccumulatorValidator` for domain-specific accumulator types.

### Integration Patterns

**On-Chain (Smart Contract Example)**:
```aiken
import lib/accumulator/types
import lib/accumulator/invariants

validator my_contract(datum: Data, redeemer: Data, ctx: ScriptContext) {
  let accumulator: AccumulatorDatum = datum     // constructor index 142 via @tag(142)
  let action: AccumulatorAction = redeemer      // constructor index 142 via @tag(142)
  
  // On-chain: validate token identity and state_digest bounds
  invariants.validate_all(accumulator, ctx)
  && my_authorization_logic(ctx)
}
```

**Off-Chain (State Monitoring Example)**:
```typescript
// Resolve descriptor from genesis transaction metadata keyed by token_name
const descriptor = await resolveDescriptor(tokenPolicyId, tokenName);

// Subscribe to accumulator state changes
const monitor = createAccumulatorMonitor({ policyId, tokenName });

monitor.on('change', async (event: AccumulatorChangeEvent) => {
  const { txHash, previousCid, newCid } = event;

  // Fetch both states from IPFS — CIDs guarantee content integrity (VERIF-1)
  const prev = await ipfs.get(previousCid);
  const next = await ipfs.get(newCid);

  // Validate anchoring and structural consistency (VERIF-2, VERIF-5)
  const isValid = await validateTransition(prev, next, descriptor);
  const diff = computeDiff(prev, next);

  console.log(`[${txHash}] ${diff.added} elements added, ${diff.removed} elements removed`);
});
```

---

## Next recommended implementation steps

### Phase 1: Core Library (On-Chain & Off-Chain Shared)

1. Initialize Aiken project and `aiken.toml`.
2. Implement `lib/accumulator/types.ak` (all core types).
3. Implement `lib/accumulator/cbor.ak` for canonical CBOR serialization/deserialization.
4. Implement `lib/accumulator/invariants.ak` with INV-1, INV-2, INV-3, INV-4, INV-5, INV-6 validators and unit tests.

### Phase 2: TypeScript Off-Chain Package

5. Initialize TypeScript package with EffectTs; define `AccumulatorObject` CBOR decoder and encoder.
6. Implement descriptor resolution from genesis transaction metadata (VERIF-6).
7. Implement IPFS integration for `AccumulatorObject` fetch (VERIF-1) and `token_name` anchoring check (VERIF-5).
8. Implement accumulator state monitor — subscribe to UTxO changes and emit `AccumulatorChangeEvent`.
9. Implement `AccumulatorValidator` interface and built-in validators (StandardMerkle, MerkleMMR, SparseMerkle, MerklePatriciaForestry) for state transition validation (VERIF-2) and diff computation.
10. Implement epoch-bounded state reconstruction: from nearest checkpoint IPFS snapshot, replay `UpdateAccumulator` redeemer operations forward; traverse epoch boundaries via checkpoint mint/burn events (VERIF-4, VERIF-7).

### Phase 3: Documentation & Examples

11. Create example validator in `examples/example_validator.ak` (demonstrates on-chain invariant checking + authorization).
12. Create TypeScript example showing accumulator state monitoring and event handling.
13. Property-based tests for built-in `AccumulatorValidator` implementations and invariant validators.

### Phase 4: Extensions (Future)

14. (Future) Rust/WASM port of the TypeScript monitoring package for performance-critical environments.
15. (Future) Checkpoint indexer — cache epoch boundary CIDs for O(1) epoch lookup in long-running accumulator chains.

---

## Open questions

- Q-01: Custom accumulator test case — define a minimal custom accumulator type to validate the `AccumulatorValidator` interface (e.g., a 2-level structure for testing).
- Q-02: Example authorization model for sample validator (PubKeyHash, NFT, multisig, or simple demo).
- Q-03: Formalize schema as a CIP after stabilization.
- Q-04: Off-chain observability API design — define the shape of `AccumulatorChangeEvent` and `StateDiff`; determine whether the monitor exposes a streaming interface, a polling interface, or both.
- Q-05: Transaction metadata schema (key 674) — define the exact CBOR structure for: (a) the genesis/minting tx entry containing `descriptor` and initial `ipfs_cid` per `token_name`; (b) spend tx entries containing `ipfs_cid` per `token_name`. The key is the `token_name` bytes.
- ~~Q-06~~: **Resolved by D-15** — `CheckpointAccumulator` (burn + remint) is the protocol's checkpoint primitive. Each checkpoint mandates a full IPFS snapshot keyed by the new `token_name`. Reconstruction within an epoch replays `UpdateAccumulator` redeemer operations from the epoch's IPFS snapshot forward, bounding replay to within one epoch.
- Q-07: Seed UTxO selection for the one-shot minting policy — should the off-chain builder be free to choose any wallet UTxO as the seed, or should the protocol mandate a specific selection convention (e.g., the first input by lexicographic `OutputReference` order)?
- Q-08: Token custody — should the `AccumulatorLifecycleToken` be co-located with the accumulator UTxO (same address, same datum) or can it be held separately (e.g., in a dedicated token UTxO at the same validator)? Co-location is simpler; separate custody could allow token-gated access patterns for external contracts.

---

## References

This file is the canonical reference for agents, design decisions, and implementation guidance for the Cardano Accumulator SDK.
