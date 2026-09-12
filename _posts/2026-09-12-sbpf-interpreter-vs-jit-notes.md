---
title: "sBPF: Interpreter vs JIT — Learning Notes"
date: 2026-09-12
excerpt: >-
  How the sBPF interpreter and JIT differ — walking through execute(), the lifetime
  transmute trick, step()'s per-instruction safety checks, verify() as static gatekeeper,
  and the JIT's register mapping and anchor-based code reuse.
---

# Difference Between Interpreter and JIT

## 1. SVM is the OS, sBPF is the CPU

- **SVM** (Solana Virtual Machine) = the OS layer: provides sandboxing, memory management, syscall dispatch, and resource limits
- **sBPF** = the CPU layer: actually executes program bytecode, with two execution modes:
  - **Interpreter**: security-first, decodes and executes instructions one at a time via `match`
  - **JIT (Just-In-Time compilation)**: performance-first, translates bytecode into native x86-64 machine code first, then runs it directly

---

## 2. Entry point `execute()`: parameter serialization + VM initialization

```rust
fn execute<'a, 'b: 'a>(
    executable: &'a Executable<InvokeContext<'static>>,
    invoke_context: &'a mut InvokeContext<'b>,
) -> Result<(), Box<dyn std::error::Error>> {
    let executable = unsafe {
        mem::transmute::<&'a Executable<InvokeContext<'static>>, &'a Executable<InvokeContext<'b>>>(
            executable,
        )
    };
    let (parameter_bytes, regions, accounts_metadata) = serialization::serialize_parameters(...)?;
    create_vm!(vm, executable, regions, accounts_metadata, invoke_context);
    let (mut vm, stack, heap) = match vm { Ok(info) => info, Err(e) => { ... } };
    let (compute_units_consumed, result) = vm.execute_program(executable, !use_jit);
}
```

What it does:

1. Serializes account/instruction data into a byte format the VM can understand
2. Creates the VM (including stack/heap)
3. Calls `vm.execute_program()`, handing control over to the "CPU layer"

---

Every reference (`&T`) in Rust carries a hidden tag indicating "how long this reference is allowed to live." The compiler uses it to prevent references from pointing at data that has already been freed (dangling pointers / use-after-free).

```rust
fn foo<'a>(x: &'a i32) -> &'a i32 { x }
// 'a is a lifetime parameter: the return value's lifetime matches the parameter's
// the compiler uses this to check that the caller won't hold onto the return value too long
```

`'static` is a special lifetime meaning "can live for the entire duration of the program" — the longest possible lifetime.

Back to the function signature:

```rust
fn execute<'a, 'b: 'a>(
    executable: &'a Executable<InvokeContext<'static>>,
    invoke_context: &'a mut InvokeContext<'b>,
) -> Result<(), Box<dyn std::error::Error>> {
```

- `<'a, 'b: 'a>`: two lifetime generic parameters. `'b: 'a` reads as "`'b` outlives `'a`," meaning `'b`'s lifespan is no shorter than `'a`'s
- Why this constraint is needed: `invoke_context: &'a mut InvokeContext<'b>` — the reference itself lives for `'a`, but the data it points to has an internal lifetime of `'b`; `'b` must be guaranteed to be longer, or the reference would dangle
- The `executable` parameter's type is bound to `InvokeContext<'static>` (the longest possible lifetime), yet inside the function body it needs to be used together with `invoke_context` (lifetime `'b`) — this is exactly why an unsafe transmute is needed later

---

Rust does memory-safety checking for you by default (borrow checking, lifetime checking, etc.). The purpose of an `unsafe` block is to tell the compiler: "I'm about to do something you can't verify the safety of on your own — I'll guarantee its safety myself."

Things `unsafe` allows that ordinary code cannot do:
- Dereferencing raw pointers
- Calling unsafe functions
- **Changing a type's lifetime annotation (via transmute)**
- Manually managing memory

Key point: `unsafe` doesn't "turn off all checks" — it means "the safety of this small block of code is now the programmer's responsibility, not the compiler's." Getting it wrong won't produce a compile error, but it can cause runtime crashes or vulnerabilities (e.g. use-after-free).

---

What is transmute

**The essence of memory: everything is just bytes.** A type is merely a label the compiler uses to interpret "how this string of bytes should be understood" — memory itself has no idea what type it is.

```
e.g. 4 bytes: 00 00 80 3F
interpreted as f32 -> 1.0
interpreted as u32 -> 1065353216
interpreted as [u8;4] -> [0, 0, 128, 63]
```

Same bytes, different "readings."

`transmute::<A, B>(x)` = **move x's bytes over unchanged, and just swap the label from A to B**. No computation, no conversion logic — pure relabeling.

```rust
let a: f32 = 1.0;
let b: u32 = unsafe { std::mem::transmute(a) };
println!("{}", b); // 1065353216, not 1!
```

Compare with a "normal conversion" using `as`:

```rust
let a: f32 = 1.0;
let b: u32 = a as u32; // result is 1, because `as` actually performs numeric conversion (truncation)
```

**This is the single most important thing to understand about transmute**: `as` "translates a value"; `transmute` "copies bytes and swaps the label." The two usually produce completely different results.

**Precondition**: the sizes must match — `size_of::<A>() == size_of::<B>()`, otherwise the compiler rejects it outright.

```rust
let x: u64 = unsafe { std::mem::transmute(1.0f32) };
// Compile error! f32 is 4 bytes, u64 is 8 bytes — size mismatch
```

**More examples to build intuition:**

```rust
// Example 1: converting between structs (works if memory layout matches)
#[repr(C)]
struct Point { x: u32, y: u32 }
#[repr(C)]
struct Pair { a: u32, b: u32 }
let p = Point { x: 1, y: 2 };
let pair: Pair = unsafe { std::mem::transmute(p) };
// pair.a == 1, pair.b == 2 — both have the same memory layout (two adjacent u32s)

// Example 2: converting a reference's lifetime (a simplified version of this article's usage)
struct Wrapper<'a>(&'a str);
fn shorten<'a, 'b>(w: &'a Wrapper<'static>) -> &'a Wrapper<'b> {
    unsafe { std::mem::transmute(w) }
}
// Wrapper<'static> and Wrapper<'b> have identical memory layout (both just store a pointer + length)
// transmute only changes what the compiler believes about "how long the inner reference can live"
// the data itself is not touched at all

// Example 3: raw pointer to function pointer (dangerous example)
let addr: usize = 0x1000;
let f: fn() = unsafe { std::mem::transmute(addr) };
// treats a raw number as a function pointer and calls it directly
// if that address doesn't hold valid code, it crashes / can be exploited
// if an attacker can control `addr`, they can redirect execution to an arbitrary address

// Example 4: violating a valid-value range (undefined-behavior example)
let x: u8 = 5;
let b: bool = unsafe { std::mem::transmute(x) };
// Undefined behavior! bool's only valid byte values are 0x00 and 0x01 — 5 is out of range
```

**Back to the line of code in the article:**

```rust
let executable = unsafe {
    mem::transmute::<&'a Executable<InvokeContext<'static>>, &'a Executable<InvokeContext<'b>>>(
        executable,
    )
};
```

- Input type: a reference to `Executable<InvokeContext<'static>>`
- Output type: a reference to `Executable<InvokeContext<'b>>`
- The bytes in memory don't change at all — only what the compiler "believes" about how long this reference can live changes

---

## 3. `execute_program`: the fork between interpreter and JIT

```rust
pub fn execute_program(
    &mut self,
    executable: &Executable<C>,
    interpreted: bool,
) -> (u64, ProgramResult) {
    let config = executable.get_config();
    if interpreted {
        let mut interpreter = Interpreter::new(self, executable, self.registers);
        while interpreter.step() {}
    } else {
        let compiled_program = executable.get_compiled_program()...;
        compiled_program.invoke(config, self, self.registers);
    }
}
```

---

## 4. The interpreter's core: `step()`

Executes exactly one instruction per call, running safety checks each time before proceeding:

```rust
pub fn step(&mut self) -> bool {
    let config = &self.executable.get_config();

    // ① Instruction metering check: has the budget run out?
    if config.enable_instruction_meter && self.vm.due_insn_count >= self.vm.previous_instruction_meter {
        throw_error!(self, EbpfError::ExceededMaxInstructions);
    }
    self.vm.due_insn_count += 1;

    // ② PC bounds check: does the current instruction address exceed the program length?
    if self.reg[11] as usize * ebpf::INSN_SIZE >= self.program.len() {
        throw_error!(self, EbpfError::ExecutionOverrun);
    }

    let mut next_pc = self.reg[11] + 1;
    let mut insn = ebpf::get_insn_unchecked(self.program, self.reg[11] as usize);

    // ③ Big match: decode and execute based on opcode
    match insn.opc {
        ebpf::RETURN | ebpf::EXIT => {
            if self.vm.call_depth == 0 {
                // this is the main program actually exiting
                self.vm.program_result = ProgramResult::Ok(self.reg[0]);
                return false;
            }
            // call_depth > 0: returning from a BPF-to-BPF call — restore registers/PC and continue
        }
        _ => throw_error!(self, EbpfError::UnsupportedInstruction),
    }

    self.reg[11] = next_pc;
    true  // continue to the next step()
}
```

### Checklist run on every `step`

| Check | Purpose |
|---|---|
| Instruction metering (`due_insn_count` vs `previous_instruction_meter`) | Prevents a program from looping forever and exhausting resources |
| PC bounds check | Prevents jumping to "wild code" outside the program body |
| Memory access (LDX/STX via `translate_memory_access!`) | Bounds + permission checks, ensuring reads/writes stay within the sandbox |
| Division/modulo | Division by zero is rejected; signed overflow cases like `i32::MIN / -1` are specifically detected |
| BPF-to-BPF CALL | `push_frame` pushes state, checks `max_call_depth`, `check_pc!` ensures the jump target is inside the code segment |
| Syscall | `dispatch_syscall` invokes the host function, updates R0 and metering |
| EXIT | Only `call_depth == 0` is a real exit; otherwise it's just a function return — restore context and continue |

Register table (12 registers; `r0`–`r9` general purpose, `r10` is the frame pointer, `pc` is a hidden register):

| Register | Purpose |
|---|---|
| r0 | Return value |
| r1-r5 | Arguments 0–4 (r5 may also serve as a stack pointer) |
| r6-r9 | Callee-saved |
| r10 | Frame pointer (system register, read-only) |
| pc | Program counter |

---

## 5. The verifier `verify()`: the static gatekeeper at load time

Unlike `step()`, `verify()` only runs **once, when loading/creating an Executable** — its job is to keep clearly-illegal bytecode from ever getting in.

```rust
fn verify<C: ContextObject>(prog: &[u8], ..., sbpf_version: SBPFVersion, ...) -> Result<(), VerifierError> {
    check_prog_len(prog)?;
    let mut insn_ptr: usize = 0;
    while (insn_ptr + 1) * ebpf::INSN_SIZE <= prog.len() {
        let insn = ebpf::get_insn(prog, insn_ptr);
        check_registers(&insn, store, insn_ptr, sbpf_version)?;
        // ... jump-bounds checks, whether call targets exist, etc.
    }
}
```

The key function `check_registers` — specifically protects **r10 (the frame pointer)** from being arbitrarily overwritten:

```rust
fn check_registers(insn: &ebpf::Insn, store: bool, insn_ptr: usize, sbpf_version: SBPFVersion) -> Result<(), VerifierError> {
    if insn.src > 10 {
        return Err(VerifierError::InvalidSourceRegister(insn_ptr));
    }
    match (insn.dst, store) {
        (0..=9, _) | (10, true) => Ok(()),
        // r10 as the target of a store instruction (writing to memory, not to r10 itself) is allowed
        (10, false) if sbpf_version.dynamic_stack_frames() && insn.opc == ebpf::ADD64_IMM => Ok(()),
        // the one exception: the dynamic-stack-frames feature allows r10 += immediate
        (10, false) => Err(VerifierError::CannotWriteR10(insn_ptr)),
        // any other attempt to directly write r10 is rejected outright
        (_, _) => Err(VerifierError::InvalidDestinationRegister(insn_ptr)),
    }
}
```

**Important boundary**: the verifier only guarantees that "this piece of bytecode is well-formed" (a static boundary) — it does **not** guarantee that "the interpreter and JIT behave identically at runtime" (runtime semantic equivalence).

---

## 6. The JIT's compiled output: `JitProgram`

```rust
pub struct JitProgram {
    page_size: usize,               // page size, used for memory alignment (setting executable permissions requires page alignment)
    pc_section: &'static mut [u32], // maps BPF instruction index -> machine-code byte offset
    text_section: &'static mut [u8],// the actual x86 machine code
}
```

**Why the `pc_section` mapping table is needed**: the machine-code length produced for a given BPF instruction isn't fixed (a simple ADD might become just one x86 instruction, while an LDX that needs bounds checking might become several), so "the Nth BPF instruction" and "the byte offset in text_section" aren't linearly related — jumps and calls must look this up via the table.

Both fields are `&'static mut`, echoing the same design rationale as `Executable<InvokeContext<'static>>` earlier — compiled machine code is meant to be cached and reused long-term.

---

## 7. The JIT's register mapping: virtual registers → real x86 registers

| eBPF register | x86-64 register | Purpose |
|---|---|---|
| R0 | RAX | Return value / scratch |
| R1-R5 | RSI, RDX, RCX, R8, R9 | Function call arguments |
| R6-R9 | RBX, R12, R13, R14 | Callee-saved |
| R10 | R15 | Frame pointer |
| Special | RDI | Pointer to the VM runtime environment |
| Special | R10 (host) | Instruction meter |
| Special | R11 | General-purpose scratch |

**Key insight**: eBPF instructions themselves **cannot directly access host registers** (e.g. they can't specify "read/write RDI" directly). This mapping layer is entirely handled by machine code that the JIT **hand-generates**; if the emitter has a bug, or the memory address-translation logic is flawed, it can lead to a sandbox escape — this is an additional attack surface that JIT introduces beyond the interpreter.

---

## 8. The JIT's compilation core: `compile()` and the anchor mechanism

### The overall flow of `compile()`

1. Calls `emit_subroutines()`, which pre-generates a batch of "shared code blocks" (anchors)
2. Main loop: iterates over every BPF instruction, records the PC → machine-code offset mapping, inserts periodic instruction-budget checks, and generates the corresponding x86 instructions based on the opcode
3. Finalization: inserts fallback exception-handling code for "unexpectedly falling off the end," resolves jump offsets (`resolve_jumps`), and seals the code section (`seal` — fills remaining space with `0xcc` debug traps, sets the memory to read-only + executable)

### Why anchors are needed: avoiding repeated generation of shared logic

**Without anchors** (regenerated everywhere):

```
Instruction 1: [division][check divide-by-zero][if so: set error code + set result slot + jump to end]
Instruction 2: [division][check divide-by-zero][if so: set error code + set result slot + jump to end]  ← duplicated
Instruction 3: [division][check divide-by-zero][if so: set error code + set result slot + jump to end]  ← duplicated again
```

Machine-code size balloons with the number of repetitions, and any future change to this logic has to be applied to every duplicate location — easy to miss a spot and end up with inconsistencies.

**With anchors** (generated once, everywhere else jumps to it):

```
[fixed location, generated only once]
ANCHOR_EXCEPTION:
  [set error code][set result slot][jump to end]

Instruction 1: [division][check divide-by-zero][if so: jmp ANCHOR_EXCEPTION]
Instruction 2: [division][check divide-by-zero][if so: jmp ANCHOR_EXCEPTION]
Instruction 3: [division][check divide-by-zero][if so: jmp ANCHOR_EXCEPTION]
```

This is essentially a "hand-implemented function reuse mechanism at the machine-code level" — because hand-written assembly doesn't have Rust's language-level abstraction of "define a function once, call it everywhere," so the developer has to build this jump-based reuse scheme manually.

```rust
fn emit_subroutines(&mut self) {
    if self.config.enable_register_tracing {
        self.set_anchor(ANCHOR_TRACE);
        // ... generates the machine code that "records register state to the trace log"
    }
    // The epilogue, unified exception entry, shared syscall logic, and other anchors are also generated here
}
```

---

## 9. The core mental model pulled together

```
verify()  ——at load time——>  only guarantees "the bytecode itself is well-formed" (static boundary)
    │
    ▼
execute() ——on every call——>  serializes parameters + creates the VM (self)
    │
    ▼
execute_program(interpreted: bool)
    │
    ├── true  → Interpreter::step() loop
    │             Interpreter = the semantic baseline/spec, pure Rust logic,
    │             safety is guaranteed naturally by the type system
    │
    └── false → runs the machine code produced by JIT compile()
                  
```
