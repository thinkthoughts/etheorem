# GETPTC THEOREM DESIGN v1

Target repository: `thinkthoughts/etheorem`  
Upstream: `etheorem/etheorem`  
Target function: `EthCLSpecs.Gloas.getPtc`  
Working branch: `proof/getptc-characterization`

## 1. Source contract being characterized

Current implementation:

```lean
forkdef getPtc (state : State) (slot : Slot) :
    StateTransition (Vector ValidatorIndex Const.ptcSize) := do
  let epoch := computeEpochAtSlot slot
  let stateEpoch := currentEpochOf state
  let spe := UInt64.ofNat Const.slotsPerEpoch
  if epoch < stateEpoch then
    let nextEpoch ← checkedAdd epoch 1 "get_ptc: epoch + 1"
    assert (nextEpoch == stateEpoch)
    pure (vmodGet (sszGet state ptcWindow) slot Const.slotsPerEpoch)
  else
    let lookaheadBound ← checkedAdd stateEpoch Const.minSeedLookahead
      "get_ptc: state_epoch + MIN_SEED_LOOKAHEAD"
    assert (epoch ≤ lookaheadBound)
    let offset ← checkedSub epoch stateEpoch "get_ptc: epoch - state_epoch"
    let windowed ← checkedAdd offset 1 "get_ptc: (epoch - state_epoch) + 1"
    let scaled ← checkedMul windowed spe
      "get_ptc: (epoch - state_epoch + 1) * SLOTS_PER_EPOCH"
    let idx ← checkedAdd scaled (slot % spe)
      "get_ptc: … + slot % SLOTS_PER_EPOCH"
    pure (vget (sszGet state ptcWindow) idx.toNat)
```

The proof ledger currently says:

- landed: both `getPtcElseOffset` bounds;
- open: a theorem about `getPtc` itself;
- desired result: a theorem carrying
  `@[characterizes EthCLSpecs.Gloas.getPtc]`.

## 2. Existing arithmetic substrate

Already proved:

```lean
theorem getPtcElseOffset_lt_next_slot [Preset] {slot curSlot : Slot}
    (hcaller : slot.toNat + 1 = curSlot.toNat)
    (hbranch : ¬ computeEpochAtSlot slot < computeEpochAtSlot curSlot) :
    getPtcElseOffset slot curSlot < 3 * slotsPerEpoch
```

and:

```lean
theorem getPtcElseOffset_lt_same_slot [Preset] (slot : Slot) :
    getPtcElseOffset slot slot < 3 * slotsPerEpoch
```

These should be reused rather than reproved inside the final contract.

---

# 3. Proposed proof decomposition

The implementation has two semantically different branches, so v1 should prefer:

```text
previous-epoch branch lemma
+
same/current-window branch lemma(s)
+
one tagged function-level characterization theorem
```

rather than one large tactic proof.

## 3.1 Previous-epoch branch

Candidate theorem shape:

```lean
theorem getPtc_prev_epoch_run_eq
    [Preset] [HasherTag]
    (state : BeaconState) (slot : Slot)
    (hprev :
      computeEpochAtSlot slot + 1 =
        currentEpochOf (pureState state)) :
    runPure
        (getPtc (StateTransition := GloasRun)
          (pureState state) slot)
        state
      =
        .ok
          (vmodGet state.ptcWindow slot Const.slotsPerEpoch,
           state)
```

### Intent

This theorem should expose the exact successful result of the
`epoch < stateEpoch` branch.

The final statement may need a slightly different hypothesis if:

- `Epoch` addition is `UInt64` and requires a no-wrap condition;
- `currentEpochOf (pureState state)` simplifies definitionally to
  `computeEpochAtSlot state.slot`;
- `runPure` or `GloasRun` uses a result/state tuple with a slightly
  different constructor shape.

### Important constraint

Do **not** silently replace the source-level checked arithmetic with
ordinary `Nat` arithmetic in the theorem statement. If a no-wrap
hypothesis is required, state it explicitly.

---

## 3.2 Same-slot/current-window branch

This is the simplest guarded caller pattern.

Candidate theorem shape:

```lean
theorem getPtc_same_slot_run_eq
    [Preset] [HasherTag]
    (state : BeaconState) :
    runPure
        (getPtc (StateTransition := GloasRun)
          (pureState state) state.slot)
        state
      =
        .ok
          (vget state.ptcWindow
            (getPtcElseOffset state.slot state.slot),
           state)
```

### Why this is attractive

For `slot = state.slot`:

```text
epoch = stateEpoch
```

so the function necessarily takes the `else` branch.

The existing theorem:

```lean
getPtcElseOffset_lt_same_slot
```

already proves the selected index lies inside the intended
`3 * slotsPerEpoch` PTC window.

### Likely refinement

The result index may need to be written in the implementation's exact
`UInt64`/`.toNat` form rather than via `getPtcElseOffset` if simplification
does not rewrite the named helper automatically.

Preferred end state:

```lean
vget state.ptcWindow (getPtcElseOffset state.slot state.slot)
```

because it connects the function-level result directly to the already
proved bound.

---

## 3.3 Next-slot caller branch

This captures the `processPayloadAttestation` caller pattern.

Candidate theorem shape:

```lean
theorem getPtc_next_slot_run_eq
    [Preset] [HasherTag]
    (state : BeaconState) (slot : Slot)
    (hcaller : slot.toNat + 1 = state.slot.toNat)
    (hbranch :
      ¬ computeEpochAtSlot slot <
        computeEpochAtSlot state.slot) :
    runPure
        (getPtc (StateTransition := GloasRun)
          (pureState state) slot)
        state
      =
        .ok
          (vget state.ptcWindow
            (getPtcElseOffset slot state.slot),
           state)
```

This should consume:

```lean
getPtcElseOffset_lt_next_slot hcaller hbranch
```

rather than reproduce its arithmetic.

### Note

The `hbranch` premise is intentionally retained because the landed
bound theorem already explains why it matters: without it, subtraction
in the `else`-branch arithmetic can underflow.

---

# 4. Tagged characterization theorem

The ledger does not merely ask for branch arithmetic. It asks for a
statement about `getPtc` itself.

Two designs are plausible.

## Design A — tagged successful-call contract

Preferred if upstream accepts a characterization theorem restricted to
the actual guarded caller patterns.

```lean
@[characterizes EthCLSpecs.Gloas.getPtc]
theorem getPtc_run_guarded
    [Preset] [HasherTag] :
    ∀ (state : BeaconState) (slot : Slot),
      (
        slot = state.slot
        ∨
        (slot.toNat + 1 = state.slot.toNat ∧
         ¬ computeEpochAtSlot slot <
           computeEpochAtSlot state.slot)
        ∨
        (computeEpochAtSlot slot + 1 =
           currentEpochOf (pureState state))
      )
      →
      ∃ ptc,
        runPure
            (getPtc (StateTransition := GloasRun)
              (pureState state) slot)
            state
          = .ok (ptc, state)
```

### Strength

This directly moves from:

```text
caller constraints
→ successful execution of getPtc
```

and can be proved by dispatching to the branch lemmas.

### Weakness

It proves successful execution and can expose exact branch values, but
it is not a complete characterization of every possible rejecting call.

---

## Design B — tagged exact branch characterization

Preferred if maintainers want the tag to describe the actual result,
not merely successful termination.

Schematic form:

```lean
@[characterizes EthCLSpecs.Gloas.getPtc]
theorem getPtc_run_eq_iff
    [Preset] [HasherTag]
    (state : BeaconState) (slot : Slot) :
    runPure
        (getPtc (StateTransition := GloasRun)
          (pureState state) slot)
        state
      =
        if computeEpochAtSlot slot <
             currentEpochOf (pureState state) then
          -- previous-epoch checked branch
          ...
        else
          -- lookahead / checked-index branch
          ...
```

### Strength

This most closely mirrors the implementation.

### Weakness

It may become unnecessarily large because it must model every
`checkedAdd`, `checkedSub`, `checkedMul`, and `assert` rejection path.

For the first PR, do not choose this design unless nearby upstream
proofs show that this exact whole-function style is expected.

---

# 5. Recommended v1 target

Start with these two theorems:

```lean
getPtc_same_slot_run_eq
getPtc_next_slot_run_eq
```

Then use them to produce one tagged theorem:

```lean
@[characterizes EthCLSpecs.Gloas.getPtc]
theorem getPtc_guarded_run_eq ...
```

where the conclusion gives the **exact PTC result**, not merely
`∃ ptc`.

Recommended semantic contract:

```text
same-slot caller
→ exact current-window PTC

next-slot caller in else branch
→ exact bounded PTC-window entry

previous-epoch successful branch
→ exact vmodGet entry
```

If all three cases can be expressed without turning the theorem into a
large checked-arithmetic error model, combine them.

Otherwise:

1. land the two guarded caller theorems;
2. ask upstream whether they consider a guarded tagged theorem
   sufficient to move the ledger row to `proved`.

---

# 6. Imports expected for the proof file

Current `GetPtc.lean` imports:

```lean
import EthCLLib.Spec.Arith
import EthCLSpecs.Gloas.EpochProcessing
```

The function-level theorem will probably also need:

```lean
import EthCLSpecs.Gloas.Operations
import EthCLLib.Proofs.Run
```

Exact imports should be minimized after the proof works.

Likely opens:

```lean
open EthCLLib.Proofs (pureState runPure)
open EthCLSpecs.Gloas
```

Do not add `Mathlib` wholesale unless the proof actually needs it.

---

# 7. Before cloning: five exact questions to answer upstream/source-side

1. Does `currentEpochOf (pureState state)` simplify directly to
   `computeEpochAtSlot state.slot`?

2. What is the exact return shape of:

   ```lean
   runPure
     (getPtc (StateTransition := GloasRun) ...)
     state
   ```

   for a successful read-only transition?

3. Is `state.ptcWindow` accepted directly by `vget`/`vmodGet`, or must
   the statement spell an SSZ projection/coercion?

4. Do existing tagged theorems characterize only valid caller
   preconditions, or is a complete reject-path equation expected for a
   throwing function?

5. Does proof coverage accept one meaningful guarded
   `@[characterizes ...]` theorem for `getPtc`, or will maintainers
   expect a whole-domain theorem?

When these five are answered, the clone gate is open.

---

# 8. Clone gate

Clone/build only after:

```text
exact statement
+ exact imports
+ exact runPure result shape
+ maintainer-compatible characterization scope
```

are known.

Until then, stay in GitHub/ZIP inspection mode so `lean-bgp212` keeps
the VPS headroom.

---

# 9. Definition of done

The first Ethereum contribution succeeds when:

```text
existing getPtc arithmetic bounds
→ function-level getPtc theorem
→ @[characterizes EthCLSpecs.Gloas.getPtc]
→ proof-coverage improvement
→ no new project axioms / sorry / admit / native_decide
→ upstream PR
```

The key result is not a new protocol.

It is:

> **the existing Ethereum Gloas `getPtc` implementation gains an explicit,
> kernel-checked Lean contract at its actual guarded call boundary.**
