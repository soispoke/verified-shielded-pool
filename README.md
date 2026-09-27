# verified-shielded-pool

A Lean 4 formal verification of the [minimal shielded pool](https://github.com/soispoke/minimal-shielded-pool), pool profile `position-notes-v2`, at commit [`8835be7`](https://github.com/soispoke/minimal-shielded-pool/commit/8835be75681dd36bfceaf73bc8c7fbd2b2c4157a).

**Status: in progress, research only.** The circuit and the pool's state machine are proven. The proof that the deployed dispatcher and settlement bytecode implement that state machine, including gas, is not done yet. The statements have been through repeated adversarial review but not yet an external human review. Do not read this repository as a claim that the pool is safe to use for anything of value.

## What is proven

[`SPEC.md`](SPEC.md) states every claim and premise in English. [`Spec/Main.lean`](Spec/Main.lean) states them in Lean. `MainTheorem` is `C1 ∧ C1c ∧ ModelTheorem ∧ ChainTheorem`.

| Result | Meaning | Status |
|---|---|---|
| `MSP.c1 : C1` | Every satisfying assignment of the pinned `spend.r1cs` (14,802 constraints) satisfies the spend relation and the compressed public signals | Proven |
| `MSP.c1c : C1c` | Every valid spend has a satisfying assignment | Proven |
| `MSP.model_theorem : ModelTheorem` | In every run of the pool model, from C1: approved spends are valid, approval cannot burn notes, no occurrence is spent twice, the pool stays solvent, credits add up, roots are real, and holders can always build a valid spend | Proven |
| `MSP.composes : Composes` | The main theorem implies the chain-level corollary | Proven |
| `MSP.c6 : C6` | The incremental tree algorithm computes the full tree's root | Proven for the Lean algorithm, not yet for the bytecode |
| `MSP.w1 : W1` | A concrete shield, publish and spend run settles with no bad event, so the claims are not vacuous | Proven |
| C2, C2c, C8, C9, C10, `Refines`, W2 | The deployed bytecode approves exactly the specified transactions, settles within its gas, and refines the model | Open |

Every claim ending in "or the run has a bad event" holds unless the run exhibits a hash collision, a degenerate hash output, a failed Groth16 extraction or a compression break among its own queries. The premises in `SPEC.md` §3 assume such events are infeasible to produce; the proofs do not establish that.

[`Proofs/README.md`](Proofs/README.md) lists every proof file. [`Artifacts/README.md`](Artifacts/README.md) explains how the pinned R1CS is imported.

## Trust boundary

The statement files are `SPEC.md`, `Proofs/AxiomAudit.lean` and every module that `Spec.lean` imports. [`STATEMENTS.lock`](STATEMENTS.lock) records their hashes and [`.github/CODEOWNERS`](.github/CODEOWNERS) assigns them to the owner. That binds once branch protection requires code owner review, which is not yet enabled. Lean's kernel checks everything else. CI rejects `sorry`, `admit`, native evaluation, new axioms and unsafe code, and `Proofs/AxiomAudit.lean` checks that each principal result depends only on `propext`, `Classical.choice` and `Quot.sound`.

## How to verify

The tools read the pool's circuit, artifacts, reference code and wallet from the parent directory, so this repository is checked out as `formal/` inside the pool at the verified commit:

```sh
git clone https://github.com/soispoke/minimal-shielded-pool
cd minimal-shielded-pool
git checkout 8835be75681dd36bfceaf73bc8c7fbd2b2c4157a
git clone https://github.com/soispoke/verified-shielded-pool formal
python3 formal/tools/check_formal.py all
cd formal
lake exe cache get
lake build
```

`check_formal.py all` checks the statement lock, the artifact hashes of `SPEC.md` §1 and the Lean sources. [The CI workflow](.github/workflows/ci.yml) also recompiles the circuit, regenerates every checked-in data file and runs the differential test of the Lean model against the pool's wallet.

## Branches

`main` is the reviewed line. `codex/formal-continuation` holds work not yet merged: a bounded proof that the pinned dispatcher bytecode rejects a transaction from the wrong sender, subgroup checks for the verification key points, and circuit mutation evidence.

`HANDOFF.md` and `CONTINUATION.md` are working notes for the agents writing the proofs.

## License

Apache-2.0, see [LICENSE](LICENSE). `Primality/` includes Apache-2.0 files from ArkLib contributors and Bolton Bailey, attributed in their headers.
