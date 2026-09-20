---
title: "x402 Solana Audit Arena
date: 2026-09-20 10:00:00 +0800
excerpt: >-
 
---

Last month I participated in the Slana audit arena, a real audit like CTF, for the first time. The first week of season 2 is a x402 agentic payment protocol, allowing an agent with an EVM wallet to pay sellers on solana. I found a valid Critical and Medium, however I didn’t score since somebody submitted it 2 hours after the mission launch and I was late, all marked as dup. This note is just for record and showing my thinking process of finding the bug.

---

I didn't start with reading the code, but trying to know what the program is doing. Once I read an accretion’s blog about how they audit MetaDao. They spend a long time analyzing if the market is really fair, could a random user gain something that the program did not intend them to gain. This reminds me that I should have something like a threaten model before diving into the code, to have a concept of what is the most vulnerable part of a system. When it comes to the x402 meridian, I was thinking about whether the seller on solana could fake the signature of the buyer and steal the money, or the buyer may fake the payment. Could the facilitator act maliciously and steal money, or attack the system. Are there any problems with the nonce?Are there any problems with the account life cycle or validation? The core part is the signature, it has to sign the right content.

Then I started to read the code, this is a very short program but I spent a while reading and understanding it because I didn’t have a concept of this kind of program. After I understood the program, I came with a more specific threat model.

1. witness check, is the witness correct

2. Signature check, has to sign the correct content.

3. Cpi check, are there any problems with the deposit and withdraw cpi

4. nonce check, are there any problems with the nonce, can the nonce be replay?

5. mint check, it use interface account, are there any problems with token-2022 extension.

6. Init check, the EVM agent user has to init a vault with it’s address as pda, are there any problems?

7. close check. Are there any problems with `close_nonce_bitmap`, rent goes to the right place?

8. EVM agents could only access via facilitator, could the facilitator act in the right way?

9. instructions sysvar check.

10. Check every utility function.

---

After I checked these I found 3 very suspicious places. The first is to use the token 2022 extension but never had a relevant check. If the buyer use a PermanentDelegate they can get the money back after the deal. I was confused about if this could be a finding， the program is only about the payment, it is the seller’s responsibility to verify the mint. In the end I saw the judge marked every finding relevant to the token 2022-extension as invalid. I also submit the finding in the end but invalid, I think this should be a info.

Then I also found 2 other problems including the critical. When I was trying to find the verification of the actual content in the signature, I only find the length check:

```rust
require!(
        data_hash.len() == message.len(),
        Permit2Error::InvalidSignature
    );
```

This definitely have some problem, the datahash bind the spender and the permit

```rust
 pub fn permit_transfer_from(
        ctx: Context<PermitTransfer>,
        permit: PermitTransferFromArgs,
        transfer_details: SignatureTransferDetails,
        eth_address: [u8; 20],
    ) -> Result<()> {
        let data_hash = hash_permit_transfer_from(&permit, &ctx.accounts.spender.key());
        permit_transfer_from_common(ctx, permit, transfer_details, eth_address, data_hash)
    }
```

 but it only verifies the length, everybody could call and set them self as the spender. It also verifies the eth address has to be correct in `verify_eth_signature`, but nothing to do with the content in data_hash

```rust
require!(
        &offsets.eth_address == eth_address,
        Permit2Error::InvalidSigner
    );
```

In this case, as long as the buyer signed any 32bytes signature, an attacker could use the signature to drain the buyer’s vault.

---

Another problem is there is a deadline, but never checks

```rust
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Copy)]
pub struct PermitTransferFromArgs {
    pub permitted_token: Pubkey,
    pub permitted_amount: u64,
    pub nonce: u64,
    pub deadline: u64,
}
```

Which means this is useless, even if it’s passed, the permit is still valid.

---


I treat solana audit arena as a great learning and practice resource since it is a full program and just like a real audit. After this audit I feel more comfortable with programs has lots of signatures and hash verifications. The ai agent on chain infrastructure will lead to new attacking vectors，I plan to dive more into x402 later, this audit CTF is very helpful.
