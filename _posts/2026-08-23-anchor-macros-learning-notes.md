---
title: "Understanding Anchor Macros"
date: 2026-08-23 09:00:00 +0800
categories: [solana, notes]
tags: [anchor, macros, proc-macro, rust, idl]
excerpt: >-
  How Anchor's proc macros turn a concise program module into the full
  Solana boilerplate — `#[program]`, `#[derive(Accounts)]`, `#[account]`,
  discriminators, and the parse-then-quote code-generation pipeline.
---

## 1. What a Native Solana Program Looks Like

At the runtime boundary, a Solana program is essentially exposed through one function:

```rust
pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult
```

The runtime gives the program three things:

- The program ID.
- The list of accounts involved in the invocation.
- A raw byte array containing the instruction data.

Without a framework, the program must handle everything else itself:

1. Inspect the beginning of the byte array to determine which method the caller wants.
2. Deserialize the remaining bytes into instruction arguments.
3. Validate every account: signer status, ownership, mutability, and other constraints.
4. Deserialize the data stored inside account buffers.
5. Execute the actual business logic.
6. Serialize modified state back into the accounts.

Most of this is boilerplate. The actual business logic may only be one or two lines long. Anchor exists to remove that boilerplate.

## 2. Anchor's Goal

Anchor aims to let developers focus on business logic:

```rust
#[program]
pub mod my_program {
    pub fn initialize(ctx: Context<Init>, data: u64) -> Result<()> {
        ctx.accounts.state.data = data;
        Ok(())
    }
}
```

The framework generates the surrounding machinery at compile time:

- Instruction dispatch.
- Argument deserialization.
- Account conversion and validation.
- State serialization and writeback.

At a high level, Anchor is a large Solana program code generator implemented with Rust procedural macros. Understanding Anchor means understanding how this code-generation system works.

## 3. Anchor's Macro System and Architecture

Anchor provides several core macros:

- `#[program]`: an attribute macro that processes the program module and generates the entrypoint and instruction dispatcher.
- `#[derive(Accounts)]`: a derive macro that processes an account-context struct and generates account parsing and constraint checks.
- `#[account]`: an attribute macro that processes account data structures and adds discriminator and serialization support.
- `declare_id!`: a function-like macro that turns a base58 program ID string into a `Pubkey` at compile time.

Anchor separates the small procedural-macro entry crates from the main parsing and code-generation implementation. Much of the heavy work lives in the ordinary `anchor-syn` crate, while the proc-macro crates act as thin wrappers.

This architecture is useful because proc-macro crates have special restrictions: they are inconvenient to use as ordinary dependencies and harder to unit test directly. Moving parsing and code generation into a normal crate makes them easier to test and allows other tools, including IDL tooling, to reuse the same logic.

This is a common architecture for substantial Rust procedural-macro projects.

## 4. Two Prerequisites: TokenStream and IDL

### TokenStream

A `TokenStream` is a sequence of Rust lexical tokens. It is the input and output format of a procedural macro and exists during compilation.

It is not yet the fully structured AST that Anchor needs. A parser such as `syn` must convert the tokens into structured Rust syntax nodes.

There are two commonly encountered types:

- `proc_macro::TokenStream`: supplied by the compiler and used in proc-macro entry-function signatures.
- `proc_macro2::TokenStream`: supplied by a third-party crate and usable in ordinary Rust crates.

They can be converted with `.into()`.

The flow is:

```text
Rust source
→ TokenStream
→ syn parser
→ structured AST
→ Anchor analysis
→ generated TokenStream
→ Rust compiler
```

### IDL

IDL means Interface Description Language. In Anchor, it is a JSON description of a program's public interface:

- Instructions.
- Instruction arguments.
- Required accounts and their order.
- Account and custom data types.
- Discriminators and other interface metadata.

A Solana program only accepts raw instruction bytes, so clients must know how to construct those bytes and which accounts to provide. Without an IDL, every client would need to reproduce those encoding rules manually.

The IDL acts as a machine-readable contract specification. TypeScript, Rust, Python, and other clients can use it to generate type-safe program calls.

With the `idl-build` feature enabled, Anchor macros generate additional IDL-related code. The build process uses that generated code to collect the complete JSON interface.

## 5. What the `#[program]` Macro Does

A procedural attribute macro has a fixed shape:

```rust
fn macro_entry(args: TokenStream, input: TokenStream) -> TokenStream
```

- `args`: tokens written inside the attribute's parentheses.
- `input`: the complete module annotated by `#[program]`.
- Return value: the token stream that replaces the annotated item.

Anchor follows a preserve-and-append strategy:

1. Preserve the user's original program module and function bodies.
2. Append generated modules and functions around it.

The generated code includes:

- `entry`: the real entrypoint called by the Solana runtime.
- `__private::__global::*`: handlers for each instruction.
- `instruction`: client-side instruction argument types and serialization support.
- `cpi`: type-safe helpers for invoking this program from another program.

Each generated instruction handler generally:

1. Deserializes instruction arguments.
2. Invokes generated account parsing and validation.
3. Calls the user's instruction function.
4. Serializes or closes modified accounts as required.

The proc-macro entry function itself does very little:

```text
Parse the input into a Program AST
→ Call to_token_stream()
→ Return the generated tokens
```

The substantial work lives in `Program`'s parsing implementation and its code-generation logic.

## 6. The Parse Phase: Understanding User Code

`impl Parse for Program` turns the user's module into Anchor's internal representation.

The process is roughly:

1. Use `syn::ItemMod` to parse the complete Rust module.
2. Traverse the module's items.
3. Select public functions as program instructions.
4. Analyze each instruction signature.

For an instruction such as:

```rust
pub fn initialize(ctx: Context<Init>, data: u64) -> Result<()>
```

Anchor extracts:

```text
Function name: initialize
Account context: Init
Instruction argument: data: u64
```

The first argument must have the expected `Context<Xxx>` form. The inner `Xxx` type connects the instruction function to the account-validation code generated by `#[derive(Accounts)]`.

The remaining arguments are instruction data: values that must be serialized by the client, sent over the network, and deserialized by the generated handler.

Anchor stores both this analyzed information and relevant original `syn` nodes inside its `Program` representation.

## 7. The ToTokens Phase: Generating Rust Code

Anchor generates Rust code with the `quote!` macro.

Three important forms are:

```rust
#value
#(#items)*
#(#items),*
```

- `#value`: insert one value's tokens into the template.
- `#(#items)*`: expand an iterable sequence without separators.
- `#(#items),*`: expand an iterable sequence with commas between items.

`format_ident!` creates valid Rust identifiers from strings.

Anchor divides code generation into focused generators:

- `entry::generate`: generates the Solana entrypoint.
- `dispatch::generate`: generates discriminator-based instruction dispatch.
- `non_inlined::generate`: generates per-instruction handlers.
- `instruction::generate`: generates instruction argument structs for clients.
- `cpi::generate`: generates the CPI module.

This separation makes each generated component easier to read and test.

An important point about `quote!` is that identifiers inside the template are literal tokens. The macro does not perform name resolution.

For example:

```rust
quote! {
    let x = ID;
}
```

The macro only emits the token `ID`. It does not know whether `ID` exists. Name resolution happens later, when Rust compiles the expanded code.

This explains why forgetting `declare_id!()` or an import can produce confusing errors inside generated code. The most useful debugging tool in that situation is:

```bash
cargo expand
```

It prints the expanded Rust code that the compiler actually sees.

## 8. The Discriminator Mechanism

Anchor traditionally uses the first eight bytes of a SHA-256 hash as a type identifier.

Instruction discriminator:

```text
sha256("global:<function_name>")[..8]
```

Account discriminator:

```text
sha256("account:<StructName>")[..8]
```

The namespace prefixes prevent the same name from representing an instruction and an account type in the same discriminator domain.

For an instruction:

```text
[8-byte discriminator][serialized arguments]
```

The generated dispatcher reads the discriminator and selects the corresponding handler.

For an account:

```text
[8-byte discriminator][serialized account fields]
```

Account deserialization checks the discriminator before decoding the fields. This prevents account data for one declared type from being silently interpreted as another Anchor account type.

Discriminators are therefore one of the foundations of Anchor's runtime type safety.

## 9. `#[derive(Accounts)]` and `#[account]`

### `#[derive(Accounts)]`

This macro processes account-context constraints such as:

- `init`, `mut`, and `signer`.
- `seeds` and `bump` for PDA derivation.
- `has_one` and `constraint` for cross-account validation.
- `close` and `realloc` for account lifecycle management.

It generates a `try_accounts` implementation that:

1. Walks through the input `&[AccountInfo]` in field order.
2. Converts each entry to the requested wrapper type.
3. Checks ownership, signer status, mutability, PDA derivation, and custom constraints.
4. Deserializes account data where required.
5. Returns the fully typed context struct.

This is what lets business logic use:

```rust
ctx.accounts.state.data = value;
```

instead of manually indexing and validating an `AccountInfo` slice.

### `#[account]`

This macro processes an account data structure and generates supporting trait implementations, including:

- `AccountSerialize`.
- `AccountDeserialize`.
- `Discriminator`.
- `Owner`, which normally identifies the current program.

These traits provide the serialization, discriminator checking, and ownership behavior required by wrappers such as `Account<T>`.

## 10. Final Mental Model

All Anchor macros perform the same fundamental task:

> At compile time, they understand concise declarative program code and generate the complete Solana boilerplate required at runtime.

The developer writes:

```rust
#[program]
pub mod my_program {
    pub fn initialize(ctx: Context<Init>, data: u64) -> Result<()> {
        ctx.accounts.state.data = data;
        Ok(())
    }
}
```

The Rust compiler actually compiles a much larger expanded program containing:

```text
Solana entrypoint
→ Instruction discriminator dispatch
→ Argument deserialization
→ Account parsing
→ Account constraint checks
→ User business logic
→ Account serialization and writeback
→ Client instruction types
→ CPI helpers
→ Optional IDL support
```

