# GETPTC THEOREM DESIGN v2

Target repository: `thinkthoughts/etheorem`  
Upstream: `etheorem/etheorem`  
Target function: `EthCLSpecs.Gloas.getPtc`  
Branch: `proof/getptc-characterization`

## Phase 0 resolution

The source-side questions are now sufficiently resolved to open the clone gate.

### 1. Plain-state / boxed-state relation

`pureState` is an abbreviation for an uncached `SSZ.Box`:

```lean
abbrev pureState {T : Type} [SSZRepr T] [HasherTag] (v : T) :
    SSZ.Box HasherTag.H T :=
  SSZ.UncachedBox HasherTag.H v
```

`currentEpochOf` is defined by reading `slot` from the boxed state and applying
`computeEpochAtSlot`:

```lean
forkdef currentEpochOf (state : State) : Epoch :=
  computeEpochAtSlot (sszGet state slot)
```

So the proof should keep the expression

```lean
currentEpochOf (pureState state)
```

until simplification/unfolding shows whether it reduces directly to
`computeEpochAtSlot state.slot`.

Do not bake that reduction into the public theorem statement unless it is
useful and stable.

### 2. Exact `runPure` success shape

Confirmed:

```lean
runPure act state : Except ε (α × BeaconState)
```

and a successful read-only run has the form:

```lean
.ok (result, state)
```

`runPure_of_run_ok` bridges from a box-level theorem

```lean
act.run (pureState v) = .ok (a, pureState w)
```

to:

```lean
runPure act v = .ok (a, w)
```

### 3. `ptcWindow` result expression

The plain Gloas `BeaconState` field is:

```lean
ptcWindow :
  Vector (Vector ValidatorIndex Const.ptcSize)
    (3 * Const.slotsPerEpoch)
```

So the value-level theorem should aim to expose the result as:

```lean
vget state.ptcWindow index
```

or:

```lean
vmodGet state.ptcWindow slot Const.slotsPerEpoch
```

rather than leaking `sszGet` into the final value-level contract.

### 4. State preservation is already proved

The repo already contains:

```lean
theorem keepsUncached_getPtc
    (state : BeaconState)
    (slot : EthCLSpecs.Gloas.Slot) :
    KeepsUncached
      (getPtc (StateTransition := GloasRun)
        (pureState state) slot)
```

Therefore the new theorem should focus on:

- successful execution under the caller guard;
- the exact returned committee;
- the branch/index relationship.

It should not reinvent a state-flavour proof.

### 5. What `@[characterizes]` requires

The attribute marks a theorem as stating a function's main contract.

Mechanically, the repo checks that:

- the target was declared by `forkdef`;
- the theorem statement actually mentions that target;
- the theorem lives under the correct fork proof directory.

Existing tagged theorems can be conditional. Therefore a guarded theorem is
compatible with the mechanism.

The human question remains whether maintainers consider the guarded contract
large enough to flip the ledger row from `in progress` to `proved`.

For `getPtc`, the ledger explicitly says the open requirement is:

> a statement about `getPtc` itself

rather than necessarily a complete theorem about every rejecting input.

This makes a guarded exact-result theorem a credible first PR target.

---

# Recommended implementation order

## A. Same-slot theorem first

This is the lowest-risk first theorem because:

```lean
slot = state.slot
```

forces the non-previous-epoch branch and the existing bound theorem already
covers its index.

Proposed shape:

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
           state) := by
  ...
```

The exact term may need a coercion/simplification adjustment after elaboration,
but this is now the intended semantic statement.

## B. Next-slot theorem

Then prove the actual payload-attestation caller case:

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
           state) := by
  ...
```

Reuse:

```lean
getPtcElseOffset_lt_next_slot hcaller hbranch
```

for the index argument.

## C. Previous-epoch theorem

Only after the else-branch contracts work, add the previous-epoch exact result:

```lean
theorem getPtc_prev_epoch_run_eq ...
```

with result:

```lean
vmodGet state.ptcWindow slot Const.slotsPerEpoch
```

and an explicit hypothesis strong enough to discharge the checked `epoch + 1`
and equality assert.

## D. Tagged contract

Finally package the meaningful caller cases into:

```lean
@[characterizes EthCLSpecs.Gloas.getPtc]
theorem getPtc_guarded_run_eq ...
```

Prefer an exact-result contract over a mere existence theorem.

---

# Important implementation detail

`getPtc` uses checked arithmetic:

```lean
checkedAdd
checkedSub
checkedMul
```

These are not decorative wrappers: they throw `.arithmetic` on overflow or
underflow.

Therefore:

- do not replace them with raw `UInt64` arithmetic without proof;
- use the caller guards and existing bounds to simplify them safely;
- preserve any genuine no-wrap premise required by the source.

This is likely the main technical work in the PR.

---

# Clone gate status

## OPEN

We now know:

- exact source function;
- exact proof file;
- exact ledger gap;
- existing arithmetic lemmas;
- existing state-flavour lemma;
- `runPure` success shape;
- value-level `ptcWindow` type;
- characterization-tag semantics;
- nearby proof style.

The next productive step is to clone the fork **only where we intend to compile
this proof**.

Because `lean-bgp212` owns VPS headroom, do not place this build on that VPS
unless resources have been freed intentionally.

---

# First implementation milestone

Do not attempt the tagged theorem first.

The first compile target is exactly:

```lean
getPtc_same_slot_run_eq
```

Success means:

```text
getPtc implementation
→ same-slot caller guard
→ checked arithmetic discharges
→ exact ptcWindow index
→ .ok (committee, unchanged state)
```

Once that theorem compiles, the rest of the PR becomes much easier to size.
