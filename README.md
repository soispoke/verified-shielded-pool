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
| `Groth16Accepts` | Textbook Groth16 verification for the pinned key, with EIP-197 encoding and the G1 and G2 subgroup checks | Defined and bound; that the deployed verifier computes it (C9) is open |
| Circuit mutations | Four mutated circuits from `SPEC.md` §6 each admit an assignment that breaks R3, R5, R7 or R8, so their C1 is false | Proven for those four (R5 only for the first input value's check) |
| C2, C2c, C8, C9, C10, `Refines`, W2 | The deployed bytecode approves exactly the specified transactions, settles within its gas, and refines the model | Open |

Every claim ending in "or the run has a bad event" holds unless the run exhibits a hash collision, a degenerate hash output, a failed Groth16 extraction or a compression break among its own queries. The premises in `SPEC.md` §3 assume such events are infeasible to produce; the proofs do not establish that.

[`Proofs/README.md`](Proofs/README.md) lists every proof file. [`Artifacts/README.md`](Artifacts/README.md) explains how the pinned R1CS is imported.

## Trust boundary

The statement files are `SPEC.md`, every module that `Spec.lean` imports, the definitions of the four mutants, `Proofs/PinClaims.lean` and the axiom audit. [`STATEMENTS.lock`](STATEMENTS.lock) records their hashes and [`.github/CODEOWNERS`](.github/CODEOWNERS) assigns them to the owner. That binds once branch protection requires code owner review, which is not yet enabled. Lean's kernel checks everything else. CI rejects `sorry`, `admit`, native evaluation, new axioms and unsafe code, and `Proofs/AxiomAudit.lean` checks that each principal result depends only on `propext`, `Classical.choice` and `Quot.sound`.

## How to verify

The tools read the pool's circuit, artifacts, reference code and wallet from the parent directory, so this repository is checked out as `formal/` inside the pool at the verified commit:

```sh
git clone https://github.com/soispoke/minimal-shielded-pool
cd minimal-shielded-pool
git checkout 8835be75681dd36bfceaf73bc8c7fbd2b2c4157a
git clone https://github.com/soispoke/verified-shielded-pool formal
git apply formal/pool-tooling.patch
python3 formal/tools/check_formal.py all
cd formal
lake exe cache get
lake build
```

[`pool-tooling.patch`](pool-tooling.patch) adds the pool's activation checks from [pool PR 24](https://github.com/soispoke/minimal-shielded-pool/pull/24) and a follow-up that strengthens them. The key tools call these checks. The patch changes only `tooling/`, not the pinned artifacts. Once the pool merges these changes, pin that commit and drop the patch.

`check_formal.py all` checks the statement lock, the artifact hashes of `SPEC.md` §1 and the Lean sources. After the first build, `lake build` is incremental and takes about a minute.

## When CI runs

[The workflow](.github/workflows/ci.yml) runs on pushes to `main`, on pull requests, weekly and on demand (Actions, "Formal verification", "Run workflow"). The pool's own CI never runs it. Pushes and pull requests reuse the project build of an earlier run and rebuild only changed modules; the weekly run and manual runs by default rebuild every module from source, which takes hours. A run that stops early saves its progress, and the next run resumes from it. A manual run can take another pool commit; `check_formal.py` then lists the pinned artifacts that differ from `SPEC.md` §1. The `formal-artifacts` job recompiles the circuit and regenerates every checked-in data file. The `formal` job builds every proof from source, audits the axioms and runs the differential test of the Lean model against the pool's wallet.

## When the pool changes

The proofs cover the pool commit pinned in `SPEC.md` §1 and in the workflow's `POOL_COMMIT`, and the pool's CI warns on pull requests that change a pinned file. To follow a new pool commit, update both pins and `STATEMENTS.lock`, then rerun the checks:

- Changes outside the pinned files (wallet, devnet scripts, docs) need nothing here.
- A change to the logic, dispatcher or verifier changes only pins today, since the chain half is open; review the Lean transcription of the tree code (C6) against the new Solidity.
- A change to the circuit is the expensive case. Regenerate the imported R1CS and data files (the tools take `--write`), then repair any circuit proofs whose wire layout moved.

## Branches

`main` is the reviewed line. `codex/formal-continuation` is an older line whose work `main` now includes.

`HANDOFF.md` and `CONTINUATION.md` are working notes for the agents writing the proofs.

## License

Apache-2.0, see [LICENSE](LICENSE). `Primality/` includes Apache-2.0 files from ArkLib contributors and Bolton Bailey, attributed in their headers.
