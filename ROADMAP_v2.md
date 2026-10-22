# ROADMAP v2 — `getPtc` Characterization

## Objective

Contribute a small, upstreamable Lean 4 proof to [`etheorem/etheorem`](https://github.com/etheorem/etheorem) by closing the tracked proof gap around:

```lean
EthCLSpecs.Gloas.getPtc
```

The goal is **not** to add a new Ethereum design, token, consensus mechanism, or application layer.

The goal is to prove a function-level contract for an existing Ethereum consensus-spec function.

---

## Why `getPtc`

`getPtc` is a good first target because:

- it is part of the Gloas consensus specification;
- its arithmetic/indexing structure is small enough to isolate;
- supporting offset-bound lemmas already exist;
- the remaining work is naturally expressed as a characterization theorem;
- it fits Etheorem's existing proof-coverage workflow;
- it gives us an actual Ethereum Lean contribution rather than a separate demonstration repo.

This follows the same development pattern that worked well for GDT:

```text
existing specification
→ isolate exact proof obligation
→ prove supporting invariants
→ characterize the actual function
→ audit dependencies
→ submit upstream
```

---

## Upstream Specification

The Gloas consensus specification defines `get_ptc(state, slot)` by:

1. computing the epoch of `slot`;
2. comparing it with the state's current epoch;
3. selecting either:
   - a prior-epoch position in `state.ptc_window`, or
   - a current/future-window position using an epoch offset;
4. returning the corresponding `PayloadTimelinessCommittee`.

The consensus-spec preconditions include the Gloas fork epoch and the permitted lookahead window.

The Lean theorem should characterize the implementation that Etheorem actually provides, not restate this Python specification independently.

---

## Existing Proof Support

Before writing the final theorem, verify the current names and signatures of the existing helper lemmas, expected to include the offset-bound results corresponding to:

```lean
getPtcElseOffset_lt_next_slot
getPtcElseOffset_lt_same_slot
```

These establish the important arithmetic fact that the computed index remains inside the PTC window for the relevant caller patterns.

Treat these lemmas as the current proof substrate, not as the final result.

---

## Target Theorem

Prove a theorem over plain `BeaconState` / input values that characterizes:

```lean
EthCLSpecs.Gloas.getPtc
```

The theorem should, where consistent with current Etheorem conventions:

- state the caller/precondition constraints explicitly;
- reason through the implementation using `runPure`;
- use existing offset-bound lemmas rather than re-proving arithmetic inline;
- identify the exact `ptc_window` index selected by each branch;
- characterize the returned committee value;
- carry:

```lean
@[characterizes EthCLSpecs.Gloas.getPtc]
```

if that remains the required proof-coverage annotation in current upstream.

### Desired proof shape

```text
slot / epoch constraints
→ branch condition
→ offset/index bound
→ valid PTC-window access
→ `getPtc` execution
→ exact returned value
```

---

## Phase 0 — No-Build Inspection

Do this before cloning or compiling on the VPS.

### Read

- `packages/EthCLSpecs/docs/PROOF_LEDGER.md`
- the current Gloas implementation containing `getPtc`;
- the existing Gloas proof file containing the offset lemmas;
- nearby successful `@[characterizes ...]` proofs;
- `CONTRIBUTING.md`;
- `CLAUDE.md`;
- relevant EthCLSpecs proof/style documentation.

### Record

Create a short working note with:

1. exact source path of `getPtc`;
2. exact proof file where the new theorem belongs;
3. current ledger row/status;
4. exact helper-lemma names and signatures;
5. two nearby characterization theorems to imitate;
6. required imports;
7. smallest build target that checks the proof.

### Clone gate

**Do not clone/build on the VPS until all seven items above are known.**

This protects resources currently reserved for `lean-bgp212`.

---

## Phase 1 — Statement Before Proof

Write the proposed theorem statement before attempting tactics.

Checklist:

- [ ] uses the actual upstream function;
- [ ] quantifies only necessary inputs;
- [ ] exposes necessary caller constraints;
- [ ] does not encode stronger assumptions than the implementation requires;
- [ ] describes the exact returned PTC value/index;
- [ ] is compatible with the proof-coverage annotation;
- [ ] is narrow enough for one PR.

If the function has multiple semantically distinct branches, prefer either:

- one theorem whose conclusion mirrors both branches cleanly; or
- two branch lemmas plus one final characterized theorem.

Avoid a giant theorem if two small branch contracts make the implementation clearer.

---

## Phase 2 — Supporting Lemmas

Reuse existing arithmetic proofs first.

Only add new helper lemmas if they are needed to bridge:

```text
epoch relation
→ offset formula
→ bounded index
```

Preferred helper properties:

- explicit;
- reusable;
- stated independently of tactic implementation;
- no duplicated proof of facts already in the repo.

Do not broaden into unrelated PTC-assignment or validator logic during v2.

---

## Phase 3 — Function-Level Characterization

Prove the actual `getPtc` contract.

Preferred strategy:

1. unfold or simplify only the necessary `runPure` / spec-function layer;
2. split on the same branch condition used by `getPtc`;
3. discharge index safety using the existing bounds;
4. simplify the selected `ptc_window` access;
5. conclude the exact returned committee;
6. attach the characterization attribute.

The proof should make the correspondence between the Lean function and the consensus-spec branch structure visible to a reviewer.

---

## Phase 4 — Verification Gates

Before opening a PR:

### Build

Run the narrowest relevant target first.

Then run the package-level target required by Etheorem's contribution workflow.

Expected examples may include:

```bash
lake build EthCLSpecs
lake build EthCLSpecsTests
```

Use the repo's current documented commands rather than assuming these remain sufficient.

### Proof coverage

Run the current proof-coverage commands documented upstream, expected to include:

```bash
just proof-coverage
just proof-coverage-check
```

Success criterion:

- the `getPtc` ledger/coverage status moves as intended;
- no unrelated proof-coverage regression occurs.

### Source audit

Check the changed Lean files for:

```text
sorry
admit
axiom
native_decide
```

Do not claim "0 axioms" for the whole monorepo: Etheorem documents explicit cryptographic trust-boundary axioms in other packages.

Our claim should be narrower:

> no new project axioms or proof holes introduced by this contribution.

### Kernel dependency audit

Run:

```lean
#print axioms <new_theorem_name>
```

Record the result in the PR description if useful.

---

## Phase 5 — Ledger and Documentation

If upstream convention requires it:

- update the `getPtc` row in `PROOF_LEDGER.md`;
- change only the status/evidence justified by the new theorem;
- do not alter unrelated rows.

Keep README/documentation edits minimal unless maintainers request more.

---

## Phase 6 — Upstream PR

### Branch

Suggested branch:

```text
proof/getptc-characterization
```

### PR scope

One proof target:

```text
EthCLSpecs.Gloas.getPtc
```

### Suggested PR title

```text
prove Gloas getPtc characterization
```

### PR body should report

- exact theorem added;
- existing helper lemmas reused;
- characterization attribute added;
- proof-coverage result;
- build/test commands run;
- `#print axioms` result;
- no new axioms / `sorry` / `admit` / `native_decide`.

Avoid broader claims about Ethereum correctness beyond the theorem actually proved.

---

## Scope Boundary

ROADMAP v2 explicitly does **not** attempt:

- a new Proof-of-Care protocol;
- Solidity verification;
- EVM execution semantics;
- validator-wide PTC assignment correctness;
- `processBuilderExitRequest`;
- `getExpectedWithdrawals`;
- SSZ proof expansion;
- SHA-256 / cryptographic trust-boundary replacement;
- Werracle integration;
- GDT integration into Ethereum.

Those may become later projects only after this first upstream proof lands.

---

## v3 Candidate Targets

After `getPtc`, reassess upstream needs rather than automatically continuing.

Candidate directions:

1. `processBuilderExitRequest` characterization;
2. `getExpectedWithdrawals` bound / characterization;
3. additional Gloas state-transition contracts;
4. only then consider a broader "specified state transition" abstraction.

A future Proof-of-Care project should emerge from repeated successful contracts, not precede them.

---

## Success Criterion

ROADMAP v2 is complete when:

```text
Etheorem ledger gap
→ exact theorem statement
→ kernel-checked characterization
→ proof-coverage update
→ clean upstream PR
```

The milestone is not "we started an Ethereum repo."

The milestone is:

> **an existing Ethereum consensus-spec function gained an upstream Lean characterization theorem.**
