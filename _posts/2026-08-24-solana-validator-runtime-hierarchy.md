---
title: "Solana Validator Runtime Hierarchy and Audit Path"
date: 2026-08-24 00:00:00 +0800
categories: [solana, notes]
tags: [solana, audit, security, svm]
excerpt: >-
  A conceptual map of the Solana validator runtime — Bank, SVM, Program Runtime, SBF VM — and where auditors should focus at each boundary.
---

> These are **conceptual architecture diagrams**, not Rust ownership trees. An arrow labeled `calls`, `uses`, or `loads through callback` represents a runtime relationship—not structural containment.

## 1. Validator Overall Architecture

```mermaid
flowchart TD
    V["Validator"]

    V --> TPU["TPU · produce blocks"]
    V --> TVU["TVU · validate blocks"]
    V --> B["Bank · slot state and orchestration"]
    V --> C["Consensus and forks"]

    TPU --> BS["Banking Stage"]
    BS -->|"schedules transactions and calls"| B

    TVU --> RS["Replay Stage"]
    RS -->|"replays transactions through"| B

    C --> BF["Bank Forks · fork choice · voting · root"]
    BF -->|"selects and roots"| B

    B -->|"owns account-state interface"| A["Accounts subsystem"]
    A --> ADB["Accounts DB"]
    ADB --> AI["Accounts Index"]
    ADB --> AC["Accounts Cache"]
    ADB --> AV["AppendVec storage"]

    B -->|"creates, configures, and calls"| SVM["SVM · TransactionBatchProcessor"]
    SVM -->|"loads accounts through callback"| A
    SVM --> AL["Account loading and transaction checks"]
    SVM --> TC["Transaction Context"]
    SVM --> PR["Program Runtime"]
    SVM --> PV["Post-execution validation"]

    PR --> IC["Invoke Context · CPI · syscalls · compute meter"]
    PR --> BP["Builtin programs · native Rust"]
    PR --> VM["SBF VM"]

    VM --> BC["Verified SBF bytecode"]
    VM --> MEM["Registers · stack · heap · input memory"]
    VM --> EXEC["Interpreter or JIT"]
    VM --> OP["On-chain program"]

    OP --> NP["Direct Solana SDK SBF program"]
    OP --> AP["Anchor-generated program"]
    AP --> DISP["Dispatcher and deserialization"]
    AP --> CONS["Accounts constraints"]
    AP --> LOGIC["Business logic"]

    SVM -->|"returns execution result"| B
    B --> CR["Commit or rollback"]
    CR -->|"persists accepted changes"| A
```

The key relationship is:

```text
Bank calls SVM
  → SVM loads accounts through a callback
  → Program Runtime invokes a builtin or the SBF VM
  → SBF VM executes on-chain program bytecode
  → SVM validates the result
  → Bank consumes the result and commits or rolls back state
```

Important corrections:

- `Bank` is **outside** the SVM; it is the validator-side caller and result consumer.
- `Accounts DB` is not inside the SVM. The SVM accesses account state through callbacks supplied by its consumer.
- `Anchor` is not part of the SBF VM. Anchor generates program code that is compiled to SBF and executed by the VM.
- TPU/TVU stages call into Bank; they are transaction pipelines, not children of the SVM.

## 2. Smart Contract Audit Path

```mermaid
flowchart TD
    TX["Sanitized transaction"]
    META["Account metadata · signer · writable · program IDs"]
    BANK["Bank environment · slot · features · fees · blockhash"]
    LOAD["SVM account loading and pre-execution checks"]
    CTX["Transaction Context · privileges · ownership · borrowing"]
    INVOKE["Program Runtime · invocation stack · CPI · PDA signers"]
    VM["SBF VM · bytecode and memory enforcement"]
    ANCHOR["Anchor-generated dispatch and account constraints"]
    LOGIC["Program business logic"]
    VERIFY["SVM post-execution validation"]
    RESULT["Transaction execution result"]
    COMMIT["Bank commit or rollback"]
    STATE["Accounts DB state"]

    TX --> META
    META --> BANK
    BANK --> LOAD
    LOAD --> CTX
    CTX --> INVOKE
    INVOKE --> VM
    VM --> ANCHOR
    ANCHOR --> LOGIC
    LOGIC --> VERIFY
    VERIFY --> RESULT
    RESULT --> COMMIT
    COMMIT --> STATE
```

### What to Audit at Each Boundary

| Boundary | Main audit questions |
| --- | --- |
| Transaction → SVM | Are account order, signer flags, writable flags, program IDs, and remaining accounts used safely? |
| Account loading | Can the wrong account, program version, fee payer, or nonce state be selected? |
| Transaction Context | Can signer/writable privileges, ownership, aliases, or mutable borrows be abused? |
| Program Runtime / CPI | Are privileges propagated correctly? Are PDA signer seeds and CPI targets verified? |
| SBF VM boundary | Are input regions, stack/heap limits, compute units, syscalls, and bytecode assumptions relevant to the bug? |
| Anchor-generated checks | Do constraints actually enforce owner, address, seeds, mint, authority, token program, close, and realloc rules? |
| Business logic | Are authorization, arithmetic, state transitions, oracle assumptions, and economic invariants correct? |
| Post-execution validation | Does runtime validation catch the mutation, and what does it deliberately not understand about business logic? |
| Commit / rollback | Which account changes survive success or failure? What happens to fees and durable nonce state? |

## Responsibility Boundary

The runtime enforces protocol privileges, ownership rules, legal account mutations, atomicity, and runtime invariants. The program must still enforce authority, PDA, token, oracle, remaining-account, business-state, and economic invariants.

In short: the runtime can prove that a state change is **protocol-valid**. It cannot prove that the state change is **correct for the application's intended business rules**.

## Official References

- [Anza Validator Anatomy](https://docs.anza.xyz/validator/anatomy)
- [Anza Runtime](https://docs.anza.xyz/validator/runtime)
- [Agave SVM Specification](https://github.com/anza-xyz/agave/blob/master/svm/doc/spec.md)
