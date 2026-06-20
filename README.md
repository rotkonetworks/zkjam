# Ligerito DA Commitment: Design Document

## Problem

A JAM end user submits a work package to a service. After 6 seconds,
the work report appears in a block with a guarantee (3 guarantor
signatures) and availability assurances (2/3 validators). The user
wants to verify: "my specific input data is in the committed work
package" — without waiting 18s for GRANDPA finality, and without
trusting a single RPC node's claim.

Today, the erasure_root in the work report is a flat Merkle tree over
1023 erasure-coded chunks. A user can't verify their input is in there
without reconstructing the entire bundle from chunks. The Merkle
proof proves chunk inclusion, not byte-range inclusion in the
original data.

## Solution

Replace the flat chunk Merkle root with a 2D tensor commitment
(Ligerito/ZODA). The tensor encoding arranges the work package
bundle as a matrix of GF(2^32) field elements, RS-encodes columns,
and commits rows via Merkle tree. The resulting root is the
erasure_root in the work report.

The tensor commitment supports **row opening proofs**: given a set of
row indices, the prover opens those rows with a batched Merkle proof.
The verifier checks the Merkle proof against the erasure_root and
recovers the original data bytes at those positions.

This gives the user a cryptographic proof that specific byte ranges
of the work package bundle are committed under the erasure_root —
the same root that 2/3 of validators attested to via assurances.

## Trust model

The user trusts:
1. The erasure_root in the block header is correct (block author signed it)
2. The guarantee (3 staked guarantors processed this work package)
3. The assurances (2/3 validators confirmed the data is available)
4. The opening proof is mathematically valid (self-verifiable)

This is strictly stronger than "trust the RPC node" — the proof
is checked locally against data from the block header. It's weaker
than GRANDPA finality — the block could theoretically be reverted.

**The pre-finality guarantee breaks under:**
- **Network partitions** (tier 1 ISP netsplit): two validator subsets
  produce competing blocks. Your work report may be in the fork that
  loses. The opening proof is valid for a block that gets orphaned.
- **Equivocation**: a block author produces two blocks for the same
  slot. Assurances may exist for both before the network resolves.

In both cases, the opening proof proves inclusion in a block that
may not survive into the canonical chain. GRANDPA finality (18s)
exists precisely to resolve these cases.

**The honest value proposition**: the opening proof provides
cryptographic data inclusion verification (trust minimization
against RPC nodes), not faster finality. Services that accept
pre-finality confirmations already trust the guarantor model —
the opening proof adds verifiability of *which data* was committed,
not stronger guarantees that the block survives.

**Latency: 6-12 seconds** (1-2 blocks for guarantee + assurances)
vs 18 seconds for GRANDPA finality. The latency improvement applies
only to services that already accept pre-finality risk.

## What this does NOT do

- **Execution verification**: the opening proof shows your data is
  committed, not that the PVM executed it correctly. Execution
  correctness comes from the guarantor trust model (3 signatures +
  auditor slashing).

- **Full zero-knowledge**: Ligerito is witness-indistinguishable,
  not zero-knowledge. Opened rows reveal data. This is fine — the
  user is verifying their OWN data inclusion.

- **Replace GRANDPA**: this is pre-finality confidence, not
  finality. Applications requiring settlement finality still wait
  for GRANDPA.

## Architecture

```
Block N: guarantor produces WorkReport
  ├── erasure_root: tensor commitment Merkle root
  ├── 1D erasure chunks distributed to validators (grey-erasure)
  └── EncodedBlock stored in GuarantorState (grey-commitment)
Block N+1: assurances confirm DA (2/3 validators)
User at N+1:
  ├── Tracks block headers (header sync)
  ├── Sees guarantee + assurances for their work package
  ├── Requests row opening proof from any node with the data
  ├── Verifies proof against erasure_root from header
  └── Confirms: "my bytes are in the committed bundle" (6-12s)
```

## Opening proof format

A row opening proof contains:
- `rows`: array of opened row data (each row = n GF(2^32) elements)
- `merkle_proof`: batched Merkle inclusion proof (sibling hashes)
- `indices`: which rows were opened

The user converts their input bytes to GF(2^32) elements, identifies
which matrix rows contain their data (from the byte offset and matrix
dimensions), and requests those specific rows.

Proof size: ~5 KB for a typical opening (a few rows + Merkle path).
Verification: hash the rows, verify Merkle proof against erasure_root.

## Encoding overhead

Per Gray Paper, max bundle size is Cmaxbundlesize = 13,791,360 bytes (~13.8MB).

| Bundle size   | GF(2^32) elems | Matrix      | Encoding time | EncodedBlock memory |
|---------------|----------------|-------------|---------------|---------------------|
| 4 KB (tiny)   | 1,024          | 32 × 32     | ~50µs         | ~16 KB              |
| 48 KB (typical)| 12,288        | 128 × 96    | ~200µs        | ~192 KB             |
| 1 MB (large)  | 262,144        | 512 × 512   | ~2ms          | ~4 MB               |
| 13.8 MB (max) | 3,547,840      | 2048 × 1732 | ~30ms         | ~55 MB              |

The encoding time is dominated by RS column FFTs in GF(2^32).
The Merkle tree is smaller than the flat 1023-chunk tree used
by the existing erasure root computation.

At max bundle size, 30ms encoding + 55MB memory per work report is
the worst case. This is per-core; a guarantor handling multiple cores
accumulates proportionally. For comparison, PVM execution for a
gas-heavy work package takes seconds — the encoding overhead is
a small fraction of total processing time.

Typical work packages have small payloads (a few KB to tens of KB),
where the overhead is sub-millisecond.

## Verification overhead

Row opening proof verification (service operator side):

| Operation              | Cost      | Notes                          |
|------------------------|-----------|--------------------------------|
| Hash opened rows       | ~5µs      | 1-5 rows × BLAKE3              |
| Merkle proof verify    | ~3µs      | ~8 levels × 1 hash per level   |
| Total verification     | **~8µs**  | Independent of bundle size      |

Proof size: ~5 KB (a few rows + Merkle path).

This is the only verification path needed for data inclusion.
The full Ligerito sumcheck proof (~411µs verify, ~130 KB proof)
exists in the crate but has no current use case in the base
protocol — it would be used for circuit satisfaction proofs
(e.g. execution verification via binius64/wimius) if that path
is developed in the future.

## Proof serving

Any node with access to the EncodedBlock (or enough DA chunks to
reconstruct it) can serve opening proofs. The proof generation is:
1. Receive request: "open rows [i, j, k] for work report H"
2. Look up EncodedBlock in store (or reconstruct from chunks, ~130µs)
3. Call `block.open_rows(&[i, j, k])` — gather rows + Merkle proof
4. Return proof

This can be served via:
- The node's JSON-RPC endpoint (simplest, available today)
- A dedicated gateway service (for high-traffic applications)
- P2P protocol extension (future, if JAM adds light client peers)

The delivery mechanism is orthogonal to the proof format. The proof
is self-verifying — any correct proof verifies against the
erasure_root regardless of who served it.

## Open questions (for review)

1. **Matrix dimensions**: currently chosen as the squarest power-of-two
   split. Should this be standardized in the protocol, or left to the
   implementation? Verifiers need to know the dimensions to interpret
   row openings.

2. **Row-to-byte mapping**: how does the user know which rows contain
   their input data? The mapping depends on the matrix dimensions and
   the bundle encoding format. Should the opening proof include the
   byte range → row mapping, or should the user compute it?

3. **Proof expiry**: DA chunks expire after some period. After that,
   the EncodedBlock can't be reconstructed. Should opening proofs be
   generated eagerly (at guarantee time) or lazily (on request)?

4. **Interaction with GRANDPA**: after finality, the state root
   provides a stronger guarantee via Merkle state proofs. Should the
   opening proof include a "upgrade path" indicator so the user knows
   to switch to state proofs after finality?

5. **Hash function**: the tensor encoding uses BLAKE3 for row hashing
   (via grey-commitment). The existing JAM protocol uses Blake2b.
   Should we align to Blake2b for consistency, or keep BLAKE3 for
   speed?

## Path to verifiable computation

The tensor DA encoding produces an `EncodedBlock` that is stored in
`GuarantorState`. This block is simultaneously:
- A DA commitment (Merkle root over RS-encoded rows) — used today
- A polynomial commitment witness (Ligerito) — available for future use

The "accidental computer" observation (Evans-Angeris 2025): the RS
encoding already performed for DA is exactly the polynomial commitment
a sumcheck-based proof system needs. This means adding verifiable
computation later requires zero changes to the encoding — only a
prover that consumes the existing `EncodedBlock`.

### What this enables

A JAM service could offer users cryptographic proof of correct PVM
execution, not just guarantor signatures. The path:

```
Today (implemented):
  Work package → tensor encode → EncodedBlock stored
  Service user → requests row opening → verifies data inclusion (~8µs)
Future (requires prover integration):
  Work package → tensor encode → EncodedBlock stored (same as today)
  PVM executes → javm produces execution trace
  Prover consumes EncodedBlock + trace → binius64 proof
  Service user → verifies execution proof (~ms)
```

The prover step uses `EncodedBlock.into_witness()` which converts the
DA encoding into the polynomial commitment witness with zero
re-encoding cost. The data was already RS-encoded and Merkle-committed
during DA — the prover just runs the sumcheck on top.

### What's needed to get there

1. **PVM opcode → binius64 gate mapping** (prototyped in wimius at
   `zeratul/crates/wimius/`). Each rv64em instruction maps to 1-3
   binius64 AND/MUL constraints via 64-bit shifted value indices.
   ADD/MUL/AND are one gate each. Branches are comparison + select.

2. **Memory consistency** via offline checking (sorted access log +
   grand product argument). ~5 constraints per memory access,
   independent of address space size.

3. **Windowed proving** for long traces. Split execution into
   windows of ~1024 steps, prove each independently, chain via
   state continuity (initial state of window i+1 = final state
   of window i).

4. **binius64 integration**. The prover/verifier (~20 crates) uses
   binary tower fields (same field family as grey-commitment's
   GF(2^32)/GF(2^128)), BaseFold + FRI for polynomial commitment,
   and GKR for constraint reduction. No cross-field bridge needed.

### Use cases that justify the proving overhead

- **Bridges**: external chains that can't slash JAM validators
  need cryptographic proof, not economic security
- **Fraud proofs**: auditor disputes backed by ZK proof instead
  of re-execution — faster resolution, no need for auditor to
  have the full state
- **High-value services**: DeFi, custody, governance where the
  attack profit could exceed the slashing penalty
- **Rollup services**: service operator proves execution to
  users who don't trust any specific validator subgroup

For the base JAM protocol, the guarantor trust model (3 signatures
+ auditor slashing) is sufficient. The verifiable computation path
is for services that need stronger guarantees.

## References

- Ligerito: https://angeris.github.io/papers/ligerito.pdf
- The Accidental Computer: https://angeris.github.io/papers/accidental-computer.pdf
- ZODA: https://eprint.iacr.org/2025/034.pdf
