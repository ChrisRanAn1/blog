---
title: "Radium cp-swap update"
date: 2026-11-22 10:00:00 +0800
---

---



- In August 2026, Raydium CP-Swap added `collect_creator_fee_permissionless`. **Any account can now trigger a pool's creator fee payout.** 


---

## 1. Background: creator fees in CP-Swap

Raydium CP-Swap is a constant-product AMM. Since 2025 it has supported an optional **creator fee**: an extra fee on every swap, booked per pool and paid out to the address stored in `PoolState::pool_creator`.

Creator fees accrue in the pool's vaults and are tracked in two counters:

```rust
pub creator_fees_token_0: u64,
pub creator_fees_token_1: u64,
```

Pools can be created through three entry points:

| Instruction | Who can call it | Who becomes `pool_creator` | Creator fee |
|---|---|---|---|
| `initialize` | Anyone | The signer | Off |
| `initialize_with_permission` | Whitelisted payers only (Permission PDA) | **Any address**, no signature needed | On |
| `initialize_v2` (Nov 2026) | Anyone | The signer | Optional |

`initialize_with_permission` exists for **launchpad migrations**. When a token graduates from a bonding curve, the launchpad program (signing as a whitelisted PDA) creates the CPMM pool and names some other address as the creator. That address is often a PDA belonging to the launchpad itself, which later splits the fees.

Originally, only the creator could withdraw creator fees, through `collect_creator_fee`, which requires the creator's signature:

```rust
#[account(mut, address = pool_state.load()?.pool_creator)]
pub creator: Signer<'info>,
```

For a PDA creator, this means the owning program must CPI into CP-Swap and sign with its seeds. In practice, the integrator controls **when** fees are collected and can account for them in the same instruction.

---

## 2. The update: anyone can collect

PR #76 (merged 2026-08-19) added a second path:

```rust
pub struct CollectCreatorFeePermissionless<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,                       // anyone

    #[account(address = pool_state.load()?.pool_creator)]
    pub creator: UncheckedAccount<'info>,           // no signature required

    #[account(
        init_if_needed,
        associated_token::mint = vault_0_mint,
        associated_token::authority = creator,
        payer = payer,
        associated_token::token_program = token_0_program,
    )]
    pub creator_token_0: Box<InterfaceAccount<'info, TokenAccount>>,
    // ... same for token_1
}
```

The handler settles the accrued creator fee (after the protocol's share) and transfers it to the creator's ATAs. The design is sound on its own terms:

- **The destination is fixed.** Funds can only go to the creator's ATA. The ATA address is derived from `(creator, token_program, mint)`, and the ATA's owner field is always the creator, no matter who pays to create it.


For a wallet creator, this is a pure convenience: a bot or a front end can "push" fees to them.

What changed is the creator no longer controls when its fees arrive. Fees can now land in its ATA at any time, outside any instruction the creator's own program is running.

---

## 3. The edge case: integrators with PDA creators

### 3.1 Who is affected

We scanned all mainnet CPMM pools with `enable_creator_fee == true` and tested whether `pool_creator` lies on the ed25519 curve. Off-curve keys are PDAs.

| Creator (PDA) | Pools | Pools with unclaimed creator fees | Owning program |
|---|---|---|---|
| `5jFLVB…` | 1,595 | 1,488 | Raydium LaunchLab (`LanMV9…`) |
| `CEFj2W…` | 534 | 534 | Unidentified fee-splitter (`BR336Q…`) |
| `F8b4Rd…` | 224 | 210 | Raydium LaunchLab |
| `qrLKab…` | 35 | 31 | Metaplex Genesis (`GNS1S5…`) |
| Others | ~200 | – | Various |
| **Total** | **2,592** | | |

### 3.2 How integrators typically collect

Both third-party integrators we looked at follow the same pattern inside a single instruction:

1. CPI into CP-Swap's `collect_creator_fee`, signing as the PDA creator. Fees land in the PDA's ATA.
2. Work out how much was received.
3. Split that amount (for example 90/10 or 60/40 between creator and platform) and transfer it out.

For example, a Genesis transaction logs:

```
Program log: CollectRaydiumCpmmFeesWithCreatorFeeV2
Program CPMMoo8L3F4NbTegBCKVNunggL7H1ZpdTHKxQB5qKP1C invoke [2]
Program log: Instruction: CollectCreatorFee
Program log: Raydium creator fee split: total=10013573, creator=6008143, protocol=4005430
```

### 3.3 What a third party can do

Anyone can call `collect_creator_fee_permissionless` on these pools first. Then:

1. The fees go to the integrator's PDA ATA **without** the integrator's program running.
2. The integrator's next collection CPI finds `creator_fees_token_0 == creator_fees_token_1 == 0`, and CP-Swap returns `NoFeeCollect`:

   ```rust
   if pool_state.creator_fees_token_0 == 0 && pool_state.creator_fees_token_1 == 0 {
       return err!(ErrorCode::NoFeeCollect);
   }
   ```

   The whole integrator transaction reverts, including its transfer-out logic.
3. Once new fees accrue, the integrator's collection succeeds again. What happens to the earlier, out-of-band amount depends on **step 2 of §3.2**:

| How the integrator measures "received" | Outcome for out-of-band fees |
|---|---|
| Full ATA balance (`ata.amount`) | Swept and split on the next successful collection. **No loss.** |
| Balance delta around the CPI | Never counted. **Stuck in the PDA's ATA indefinitely**, unless the program has a separate sweep instruction. |




## 4. Takeaways

mainnet scan found that 2,592 of the 9,641 CPMM pools with the creator fee enabled have a PDA as creator, mostly owned by Raydium LaunchLab, Metaplex Genesis. Also have many which were close source.

- Assume fees can arrive in your ATA at any time, not only during your own CPI.
- When splitting, use the **full ATA balance**, or keep your own ledger and add an explicit sweep instruction.
- Handle `NoFeeCollect` gracefully so a front-run cannot block unrelated logic in the same instruction.

