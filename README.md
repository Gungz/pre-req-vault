# pre-req-vault — Turbin3 Builders Cohort prerequisite (Part 1)

An Anchor program implementing a per-user SOL vault, extended with a
**Cross-Program Invocation to Turbin3's registration program** on `withdraw`.

**Deployed program (devnet):** `6i1kTZWFVtxE412DoGz3CXB24T7KX6bZzWKxudKwASoZ`
([Explorer](https://explorer.solana.com/address/6i1kTZWFVtxE412DoGz3CXB24T7KX6bZzWKxudKwASoZ?cluster=devnet))

Architecture: [`docs/architecture.svg`](docs/architecture.svg) · explainer:
[`docs/architecture.md`](docs/architecture.md) · diagram source:
[`docs/architecture.mmd`](docs/architecture.mmd) · video: _add your YouTube link here_

---

## What was changed (vs. the original Turbin3 template)

| File | Change |
|---|---|
| `programs/pre-req-vault/src/instructions/withdraw.rs` | Implemented the TODO: after the vault→user SOL transfer, the instruction CPIs the registration program's `initialize(github)` with `{ user, application_account, system_program }` |
| `programs/pre-req-vault/src/lib.rs` | `withdraw(ctx, amount, github)` — new `github: String` argument, forwarded to the handler |
| `tests/pre-req-vault.ts` | Passes `github` to `withdraw`; **asserts on-chain** that the `prereqs` PDA recorded the same GitHub handle; added the 3 missing `await`s on `confirmTx` (run reliability) |
| `Anchor.toml`, `programs/pre-req-vault/src/lib.rs` | New program ID `6i1k…ASoZ`. The template's address `DBoobRVqT7PaAhq8obqo95723CHyWvcCLRUKvQ5wYWE4` is already deployed on devnet with Turbin3's upgrade authority, so per the challenge ("deploy the modified program, not the one we deployed") a fresh program keypair was generated (`anchor keys sync` + rebuild) |

The CPI design notes:

- the `vault` PDA owns no private key — it signs its own solvency transfer via
  **signer seeds** `["vault", vault_state.key(), vault_bump]`;
- `application_account` is derived as `["prereqs", user]` with
  `seeds::program = application_program.key()`, pinning derivation at the real
  TRBZ program rather than our own;
- the `user` signer privilege of the top-level transaction propagates through
  the CPI, so the registration program validates a legitimately signed owner.

## How to run

Prerequisites: Rust stable (rust-toolchain.toml pins it), Solana CLI (Agave) 3.x,
Anchor CLI 0.32.1, Node ≥ 20, pnpm (e.g. `corepack enable pnpm`).

```bash
pnpm install

anchor build                                  # produces target/deploy/pre_req_vault.so
anchor deploy --provider.cluster devnet --no-idl
anchor test --skip-build --skip-deploy        # runs tests/pre-req-vault.ts against devnet
```

Notes

- `Anchor.toml` is set to `cluster = "devnet"` + `skip_local_validator = true`,
  i.e. the test suite executes real transactions on devnet with the configured
  keypair (`~/.config/solana/id.json`); it needs SOL for fees, ~1 SOL for the
  deposit leg and ~1.5 SOL for program deploy rent.
- `--no-idl` sidesteps a known anchor CLI 0.32 ↔ anchor-lang 1.x IDL-upload
  mismatch; it is harmless — the TS client reads the IDL from local
  `target/idl/`.
- **One registration per wallet**: the withdraw test registers the wallet once
  with the TRBZ program and cannot be re-run on the same wallet afterwards
  (`initialize` errors "already in use"). The initialize/deposit/close legs can
  be re-run freely; run the suite with `pnpm exec ts-mocha … -g "Close"` style
  filters if you need individual legs.
- The Litesvm Rust test (`programs/pre-req-vault/tests/test_initialize.rs`)
  ships commented out in the template and is untouched; the challenge focuses
  on the TypeScript suite.

## Verification of the registered GitHub handle (success criteria)

`anchor test --skip-build --skip-deploy` — **4 passing** on devnet:

| Leg | Transaction |
|---|---|
| Initialize | [`ShvAZtziaHTSswjfthdfHGeWyZfcgybwkwLMjq1yeYpEBNsh2d4yWyA4xob7PvsqtSnWZQFNuKwNebMQ6SRj2o8`](https://explorer.solana.com/tx/ShvAZtziaHTSswjfthdfHGeWyZfcgybwkwLMjq1yeYpEBNsh2d4yWyA4xob7PvsqtSnWZQFNuKwNebMQ6SRj2o8?cluster=devnet) |
| Deposit 1 SOL | [`22QftkRtaAeDfP5U3c9W8CNjrwF46VhhzyLqauMvJPKBmQ19isZMkT1q3m2rBgMPVpj877B1hDxa4LLhdqidTEwA`](https://explorer.solana.com/tx/22QftkRtaAeDfP5U3c9W8CNjrwF46VhhzyLqauMvJPKBmQ19isZMkT1q3m2rBgMPVpj877B1hDxa4LLhdqidTEwA?cluster=devnet) |
| **Withdraw + registration CPI** | [`5D2VPxwHofw7gpVZ3oE8Bq67VSP7cwSBh41a4L126LwVTgTGeYcqY1yPRnMZkbHu2Fop9rtk5jo3a6ed7QvRdkm3`](https://explorer.solana.com/tx/5D2VPxwHofw7gpVZ3oE8Bq67VSP7cwSBh41a4L126LwVTgTGeYcqY1yPRnMZkbHu2Fop9rtk5jo3a6ed7QvRdkm3?cluster=devnet) |
| Close | [`5EBDVj56LzgEsqTqiR8SaidzPrQxNRpgZTDt7oHdaADUV5QDhUUUSdk151vidmRUZHTJfRemPyrwLqJthEoneCDp`](https://explorer.solana.com/tx/5EBDVj56LzgEsqTqiR8SaidzPrQxNRpgZTDt7oHdaADUV5QDhUUUSdk151vidmRUZHTJfRemPyrwLqJthEoneCDp?cluster=devnet) |

Inside the withdraw transaction logs you can see the nested CPI:

```
Program 6i1kTZWF…ASoZ invoke [1]
Program TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM invoke [2]   ← registration program
```

On-chain state after the run — the `prereqs` PDA owned by Turbin3's program:

| Field | Value |
|---|---|
| Address | `4iWoRFvBP8Dkxjqh7kauJ8t1vXmSYWszdRwfaTkHxj3U` ([Explorer](https://explorer.solana.com/account/4iWoRFvBP8Dkxjqh7kauJ8t1vXmSYWszdRwfaTkHxj3U?cluster=devnet)) |
| Owner program | `TRBZyQHB3m68FGeVsqTK39Wm4xejadjVhP5MAZaKWDM` (Turbin3 `q3_pre_reqs_rs`) |
| `user` | `6UBWeY293oqa98E5chTJcjZdqwqJL4Y93o56gep2HBMy` (test wallet) |
| `github` | **`Gungz`** |
| `bump` / `pre_req_ts` / `pre_req_rs` | `255` / `false` / `false` (the `pre_req_*` flags are set by Turbin3's own submit flow later) |

Re-verify it yourself:

```bash
solana program show 6i1kTZWFVtxE412DoGz3CXB24T7KX6bZzWKxudKwASoZ --url devnet
solana account 4iWoRFvBP8Dkxjqh7kauJ8t1vXmSYWszdRwfaTkHxj3U --url devnet

# decode github from ApplicationAccount (8 disc + 32 pubkey + bump + 2 bools + u32 len)
node -e "const {Connection,PublicKey}=require('@solana/web3.js');
new Connection('https://api.devnet.solana.com','confirmed').getAccountInfo(new PublicKey('4iWoRFvBP8Dkxjqh7kauJ8t1vXmSYWszdRwfaTkHxj3U')).then(i=>{const d=i.data,l=d.readUInt32LE(43);console.log('github:',JSON.stringify(d.subarray(47,47+l).toString()))})"
```

## Repository layout

```
programs/pre-req-vault/src/   anchor program (lib.rs, state.rs, instructions/{initialize,deposit,withdraw,close}.rs)
idls/registration.json        Turbin3 q3_pre_reqs_rs IDL consumed by declare_program!(registration)
tests/pre-req-vault.ts        devnet test suite (initialize → deposit → withdraw+CPI → close)
docs/                          architecture.svg/.mmd/.md + video script
```
