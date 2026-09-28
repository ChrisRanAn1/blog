---
title: "Learning to Formally Verify Solana Programs"
date: 2026-09-27 10:00:00 +0800
---

I've recently been studying [Crucible](https://blog.asymmetric.re/introducing-crucible-an-invariant-fuzzing-framework-for-solana/), Asymmetric Research's invariant-fuzzing framework for Solana. That brought me back to [OtterSec's research on formally verifying Solana programs](https://osec.io/blog/formally-verifying-solana-programs/) I learned couple weeks ago. Both techniques search for violations of security properties, but they offer different kinds of assurance.

T

## 1. Formal Verification vs. Invariant Fuzzing

Both approaches look for **counterexamples**: inputs or execution sequences that violate a property. Suppose a withdrawal instruction must reject requests exceeding the available balance:

```rust
// Illustrative requirement, not a complete withdrawal implementation.
if amount > balance {
    return Err(InsufficientBalance);
}
```

**Invariant fuzzing** generates concrete inputs, executes the program, and checks invariants. Coverage-guided fuzzing favors inputs that discover new behavior. **Stateful invariant fuzzing** also mutates *instruction sequences*, allowing it to find bugs that require multiple state transitions.

**Formal verification (FV)** models program behavior and the negation of a desired property as constraints, then asks a solver whether a counterexample exists. If the counterexample formula is unsatisfiable, and the model and verification scope are adequate, the result is a proof **within that scope**—not merely a report that testing has not found a bug.


## 2. From Symbolic Execution to Bounded Model Checking

### 2.1 Concrete vs. symbolic execution

Concrete execution starts with actual values:

```text
a = 10
b = 20
x = a + b = 30
```

Symbolic execution retains the relationship `x = a + b` without selecting particular values for `a` and `b`. To check `assert(x != 100)`, a solver searches for a solution to the counterexample condition `a + b = 100`. If one exists, the assertion is not universally valid over those inputs.

### 2.2 Branches become path constraints

Consider:

```rust
let y = x + 3;
let z = if y > 100 { y * 2 } else { y + 1 };
assert!(z != 105);
```

Symbolic analysis represents the two branches:

- Path A: `y = x + 3 ∧ y > 100 ∧ z = 2y`.
- Path B: `y = x + 3 ∧ y <= 100 ∧ z = y + 1`.

To look for a violation, add the negated assertion: `z = 105`. Neither path is satisfiable under mathematical-integer semantics: Path A would require `y > 100` and `2y = 105`; Path B would require `y <= 100` and `y + 1 = 105`. Actual fixed-width integer behavior must be modeled separately, including overflow.

### 2.3 What SAT and UNSAT mean

When checking *program constraints AND the negated property*:

- **SAT:** a satisfying assignment exists—a potential counterexample that must be interpreted against the model.
- **UNSAT:** no counterexample exists in the correctly encoded, covered scope.
- **Unknown / incomplete:** the solver did not establish either result. A timeout is not a proof.

A minimal Z3 example:

```python
from z3 import *

x, y, z = Ints('x y z')
s = Solver()
s.add(y == x + 3, y > 100, z == y * 2)
s.add(z == 105)  # Negation of assert(z != 105)
print(s.check())  # unsat
```

### 2.4 Why bounded model checking?

Independent binary branches can create up to `2^n` paths. Symbolic loop bounds introduce another challenge: the verifier may not know how many iterations to explore.

**Bounded model checking (BMC)** unfolds loops up to a selected bound. For a bound of three iterations, it considers executions with zero through three iterations. Proving safety within that bound does **not** establish safety for arbitrary execution lengths. If the verifier can also prove that a fourth iteration is impossible for the modeled inputs, then the bound covers those loop executions completely.

## 3. Kani and CBMC: The Rust Verification Stack

OtterSec uses **Kani** as the Rust-facing verification tool and **CBMC** as the underlying bounded model checker. At a high level:

```text
Rust program + proof harness
             |
             v
           Kani
       Rust translation
             |
             v
           CBMC
     Constraint encoding
             |
             v
         SAT solver
```

Kani provides a convenient interface for symbolic inputs, assumptions, and proof assertions:

```rust
fn add_one(x: u8) -> u8 {
    x + 1
}

#[kani::proof]
fn verify_add_one() {
    let x: u8 = kani::any();
    kani::assume(x < 255);
    let result = add_one(x);
    assert!(result > x);
}
```

- `#[kani::proof]` marks a **proof harness**.
- `kani::any()` creates an arbitrary, type-constrained symbolic input.
- `kani::assume(...)` limits the verification domain.
- `assert!(...)` states the property to prove.

The `x < 255` assumption matters: without it, adding one to a `u8` can overflow. Conversely, an overly restrictive `assume` can exclude genuinely reachable bugs. A proof is only as useful as its assumptions.

## 4. Specifying a Solana Program

This is the central step in applying FV to Solana: **turn security expectations into precise properties that a checker can evaluate**. OtterSec distinguishes instruction specifications from account invariants.

### 4.1 Instruction specifications: `errors_if` and `succeeds_if`

An instruction specification describes when an instruction **must fail** or **must succeed**.

```rust
#[errors_if(balance < amount)]
```

This means:

```text
balance < amount  =>  Error
```

If an insufficient-balance withdrawal succeeds, the specification is violated. It does **not** say that a withdrawal with sufficient funds must succeed; authorization or other requirements may still cause an error.

The corresponding positive condition is:

```rust
#[succeeds_if(balance >= amount)]
```

This requires success whenever the specified premise holds, without specifying what happens outside that premise. A crank instruction intended to advance the protocol in every valid state may motivate `succeeds_if(true)`, but its environment and valid-state assumptions still need to be modeled correctly.

Together, complementary specifications can describe an exact success condition:

```rust
#[succeeds_if(P)]
#[errors_if(!P)]
```

They require `Success ⇔ P` in the model. In practice, fully defining every success condition may be unnecessarily expensive; a narrower, meaningful one-way property is often preferable.

### 4.2 Account invariants

An **account invariant** describes which states an account is allowed to occupy, rather than only whether a function returns `Ok` or `Err`.

For example:

```rust
#[invariant(self.assets >= self.liabilities)]
struct UserStatement {
    owner: Pubkey,
    assets: u64,
    liabilities: u64,
}
```

Successful relevant operations must preserve `assets >= liabilities`.

For a Squads multisig account, structural requirements include:

```rust
#[invariant(
    !self.keys.is_empty()
    && self.keys.len() <= u16::MAX as usize
    && self.threshold >= 1
    && (self.threshold as usize) <= self.keys.len()
)]
```

These require at least one member, a representable member count, at least one required approval, and no more required approvals than available members.

## 5. Turning Specifications into Proof Harnesses

Define:

- `P0`: relevant account invariants before instruction execution.
- `P1`: relevant account invariants after execution.
- `K`: the instruction returns success (`Ok`).
- `S`: the premise of `succeeds_if`.
- `E`: the premise of `errors_if`.

OtterSec's three proof obligations address distinct questions.

### 5.1 Account invariant: does success preserve valid state?

```text
(P0 ∧ K) => P1
```

Illustrative harness:

```rust
assume(P0);
let res = instruction_handler(...);
assert!(!is_ok(&res) || P1);
```

**A successful instruction must not turn a valid account into an invalid one.** A failed Solana transaction normally rolls back state changes; this particular implication does not require checking the post-state of a failed instruction. It also does not prove liveness: a function that always returns an error could satisfy this obligation vacuously.

### 5.2 Positive instruction: must valid inputs succeed?

```text
S => K
```

```rust
assume(S);
let res = instruction_handler(...);
assert!(is_ok(&res));
```

If a withdrawal requires a signer, specific account permissions, or other environmental conditions, those requirements must be represented in the success premise or verification environment.

### 5.3 Negative instruction: must invalid inputs fail?

```text
E => ¬K
```

```rust
assume(E);
let res = instruction_handler(...);
assert!(!is_ok(&res));
```

For example, an insufficient-balance withdrawal must never succeed.

| Proof obligation | Question answered |
|---|---|
| Account invariant | Does every successful relevant operation preserve valid account state? |
| Positive instruction | Does the instruction succeed whenever the stated success premise holds? |
| Negative instruction | Does the instruction fail whenever the stated failure premise holds? |

**These proofs are not interchangeable.** In particular, proving a function rejects bad inputs is different from proving its successful executions cannot create bad account states.

## 6. Squads Case Study: Refining Specifications with Counterexamples

The Squads examples show a practical FV workflow: start with an explicit claim, inspect the solver's counterexample, and decide whether it exposes a program defect or an incomplete specification.

### 6.1 Creating a multisig: refine an overly broad success claim

OtterSec initially tries to prove that the multisig `create` instruction always succeeds:

```rust
#[succeeds_if(true)]
```

This fails with a counterexample such as `threshold = 33764` for `19` members. That does not establish a program bug. The program is supposed to reject an impossible signing threshold; **the specification was too broad**.

Refinement proceeds in stages:

1. Require `threshold <= members.len()`. The next counterexample uses `threshold = 0`.
2. Require `threshold != 0`. Another counterexample has an excessively large member count (`536870920`).
3. Constrain `members.len() <= u16::MAX as usize`.

The resulting success specification is:

```rust
#[succeeds_if(
    threshold != 0
    && (threshold as usize) <= members.len()
    && members.len() <= u16::MAX as usize
)]
```

Equivalently:

```text
1 <= threshold <= members.len() <= 65535
```

This specification passes in the paper's verification setup. A separate `members.len() >= 1` condition is redundant: it already follows from the first two inequalities.

**Practical lesson:** when a positive proof fails, first ask whether the counterexample should legitimately be accepted by the program. Tighten assumptions only to exclude genuinely invalid or unreachable situations—not just to make a proof pass.

### 6.2 `SparseVec`: model only what the property needs

Some of the `create` checks depend only on the number of members, not their concrete public keys. A lightweight model can therefore represent the vector as:

```text
members = SparseVec { size: 19 }
```

This reduces the number of symbolic elements and data-structure operations the solver must reason about. The simplification has a boundary: properties involving element identity, lookup, sorting, or deletion need a model that preserves those semantics. A smaller model is useful only when it remains sound for the property being proved.

### 6.3 Verifying the multisig threshold invariant

The next question is different: **if `create` succeeds, is the newly created multisig account valid?** Its threshold requirement is:

```text
1 <= threshold <= keys.len()
```

OtterSec deliberately removes the implementation's threshold validation and reruns verification. The solver exposes a state such as:

```text
threshold = 32768
keys.size = 5112
create() = Ok
```

Here, the program *should have rejected* the invalid creation request but instead successfully created an invalid account. This violates the **account invariant**:

```text
(P0 ∧ K) => P1
```

This differs fundamentally from the earlier failed `succeeds_if(true)` proof. The earlier counterexamples showed appropriate rejection of invalid inputs; this counterexample shows inappropriate acceptance leading to invalid state.

### 6.4 `remove_member`: distinguishing invalid states from real bugs

For member removal, the paper examines these requirements:

```rust
#[succeeds_if(keys.len() > 1)]
#[errors_if(keys.len() <= 1)]
```

The negative proof initially fails with a zero-member multisig. The instruction explicitly rejects `keys.len() == 1`, but zero members can bypass that check; the underlying removal operation finds no member and can still return success.

The key question is whether a zero-member multisig can actually be a **reachable initial state**.

If account creation and every relevant state-changing operation have been proved to preserve:

```text
1 <= threshold <= keys.len()
```

then a zero-member account is not reachable from valid initial state. Restoring the account invariant as a precondition excludes this spurious counterexample.

This illustrates the **two roles of an account invariant**:

- **Postcondition:** successful operations must preserve it.
- **Precondition:** once initialization and all relevant transitions have been verified, subsequent proofs may assume it holds on entry.

An unproved invariant must not be used to dismiss a genuinely reachable failure.

## 7. What If a CPI Cannot Be Fully Verified?

Cross-program invocations (CPIs) complicate static reasoning: the callee's implementation, return values, and deeper call chain may be outside the current verification model. Proving local Rust logic is not automatically a proof of the entire instruction.

OtterSec discusses a complementary approach: **check critical account invariants at runtime** when a complete static proof is impractical. For example, the following is illustrative pseudocode:

```rust
invoke_signed(...)?;

if invoke_result == 5 {
    ctx.my_account.put_into_bad_state();
}

assert!(ctx.my_account.invariant());
Ok(())
```

If the account becomes invalid, the assertion aborts execution rather than letting that path successfully commit the invalid state. A real implementation must correctly handle CPI results, account borrowing, error semantics, and on-chain computation costs.

FV attempts to establish a property *before deployment* over its modeled scope. A runtime assertion checks a property *during actual execution*. They complement one another, but a runtime assertion covers only the property and program locations where it is actually enforced; it does not establish that an external program is safe.

## 8. Practical Challenges Specific to Solana

### 8.1 Constraint explosion and lightweight SDK models

Ordinary Solana SDK types can produce large verification problems. Even `Vec<Pubkey>::contains` can involve many byte comparisons and branches when the keys and collection are symbolic.

The paper explores verification-oriented replacements, including smaller modeled public keys and simpler or fixed-size vector representations. These can reduce solving costs, but introduce a **soundness obligation**: if a replacement loses a behavior relevant to the safety property, a successful proof may not transfer to the real program.

### 8.2 Runtime environment and serialization

Solana stores account data as bytes. Anchor deserializes those bytes into Rust structures, executes application logic, and serializes updated state back into bytes:

```text
Account bytes
    |
    v  deserialize
Rust account struct
    |
    v  execute business logic
Updated Rust struct
    |
    v  serialize
Account bytes
```

Proving business logic over the Rust structure does not automatically cover every behavior of serialization, account resizing, or custom account representations. Any claim about program safety must specify which runtime operations were modeled faithfully and which were abstracted or assumed.

### 8.3 External failures and CPI call chains

Suppose program A calls B, which calls C. If C fails and the error propagates through B, A may fail even when A's local logic is correct. A broad `succeeds_if(true)` cannot be proved without accounting for these downstream behaviors and relevant environmental assumptions. As with all formal proofs, the system boundary matters.

## 9. Conclusion: A Practical Workflow for Solana FV

The most useful insight from the Squads case study is that **formal verification starts with specification**. The solver can prove only the property we express, for the states and behavior our model includes. A workable approach is:

1. **Define the safety boundary.** Identify the instruction, affected accounts, valid initial states, and external dependencies. Decide which behaviors the current model covers.
2. **Write explicit specifications.** Use `errors_if` for inputs that must fail, `succeeds_if` for conditions that must succeed, and account invariants for states that successful operations must preserve.
3. **Generate and run proof harnesses.** For account state, prove `(P0 ∧ Success) ⇒ P1`; separately prove positive and negative instruction requirements where they matter.
4. **Inspect every counterexample.** Determine whether it identifies an actual implementation bug, an overly broad success specification, an invalid assumed initial state, or a modeling limitation. Refine the relevant part—never merely suppress the counterexample.
5. **Keep abstractions and assumptions defensible.** Verify initialization and relevant state transitions before reusing account invariants as preconditions. Ensure lightweight SDK models preserve the semantics the proof depends on.
6. **Address what remains unproved.** Model CPI and runtime behavior where practical; use targeted runtime assertions when static verification is incomplete, and document the remaining limitations.


---

### Reference

- [OtterSec — Formally Verifying Solana Programs](https://osec.io/blog/formally-verifying-solana-programs/)
