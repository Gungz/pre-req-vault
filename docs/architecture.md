# Architecture — pre-req-vault

> A primer for a new engineer joining the project. Program ID on devnet:
> `6i1kTZWFVtxE412DoGz3CXB24T7KX6bZzWKxudKwASoZ` (source: `programs/pre-req-vault/`).
> Diagram: [`architecture.svg`](./architecture.svg) (source: [`architecture.mmd`](./architecture.mmd)).

## 1. What this program is

A minimal **SOL escrow** written with Anchor. Each wallet owns exactly one private
vault: you initialize it, push SOL in with `deposit`, pull SOL out with `withdraw`,
and tear it down with `close`. The one non-obvious part: **`withdraw` also
cross-program-invokes Turbin3's registration program** to record the application
details on-chain (this repository's prerequisite challenge).

## 2. Accounts at a glance

| Account | Type | Seeds (PDA derivation) | Role |
|---|---|---|---|
| `user` | `Signer`, funded | normal keypair (`6UBWeY…HBMy` on devnet) | pays every fee + rent, owns the flow |
| `vault_state` | `VaultState` (init'd by our program) | `["state", user.key()]` | stores the vault PDA's bump + its own bump |
| `vault` | system-owned lamport holder | `["vault", vault_state.key()]` | holds the SOL; **no data**, exists to receive lamports |
| `application_account` | UncheckedAccount (foreign) | `["prereqs", user.key()]` **at the TRBZ program** | created by the CPI; stores `{ user, bump, pre_req_ts, pre_req_rs, github }` |
| `application_program` | program identity | — | guards that the CPI executes against `TRBZyQHB…aKWDM`, not an impostor |
| `system_program` | native program | — | lamport transfers & account creation |

PDA rule of thumb: `vault` is derived *from the `vault_state` address* (not from the
user), so both bumps are stored once in `VaultState` and re-used to reconstruct the
signer seeds on every withdrawal.

```rust
// state.rs
pub struct VaultState { vault_bump: u8, state_bump: u8 }   // 8 + 2 bytes on chain
```

## 3. Instruction walkthrough

```
initialize            deposit(amount)        withdraw(amount, github)          close()
     │                      │                           │                          │
     ├── creates the        ├── System CPI:             ├── System CPI:            ├── drains vault → user
     │   VaultState PDA     │   user → vault            │   vault → user           ├── closes VaultState
     │   [state, user]      │   (vault account is       │   (vault signs via       │   (rent refund)
     ├── stores the bumps   │   created on 1st deposit) │   its seeds + bump)      │
                            │                            │                           │
                            │                            └── CPI to TRBZ program:
                            │                                initialize(github)
                            │                                → creates prereqs PDA,
                            │                                  records the username
```

### The interesting instruction: `withdraw`

1. **SOL out first.** A `system_program::transfer` CPI moves `amount` lamports from
   the `vault` PDA to the user. `vault` has no private key, so the program signs
   *for* it with **signer seeds**: `["vault", vault_state.key(), vault_bump]`.
2. **Then the registration CPI.** `declare_program!(registration)` pulls
   Turbin3's `q3_pre_reqs_rs` interface from `idls/registration.json`; the instruction
   `registration::cpi::initialize(github)` is invoked with
   `{ user, application_account, system_program }`. Two things make this safe:
   - the `Withdraw` accounts struct derives `application_account` with
     `seeds::program = application_program.key()`, pinning the fork at the **real
     TRBZ program**, and
   - `user` keeps its top-level-transaction signer flag through the CPI
     (signer privileges propagate across invokes), so Turbin3's program sees a
     legitimately signed owner.
3. Turbin3's program creates the `prereqs` PDA and stores the GitHub username and
   which pre-reqs are done. **One registration per wallet** — the created account
   persists even after our vault is closed.

## 4. State lifecycle

```
un-initialized ──initialize──► READY(empty) ──deposit──► FUNDED ──withdraw──► ~partial~ ──...──┐
                                   ▲                                                           │
                                   └────────────── close() ends the round-trip ────────────────┘
(drop to zero, rent refunded, vault_state closed; prereqs registration stays on devnet forever)
```

`close` doubles as the final withdrawal: it drains whatever remains in `vault`.

## 5. How to run it locally

```bash
pnpm install
anchor build
anchor deploy            # devnet, per Anchor.toml (--no-idl avoids the 0.x IDL upload quirk)
anchor test --skip-build --skip-deploy   # runs the anchors test suite against devnet
```

TS suite: `tests/pre-req-vault.ts` — initialize → deposit(1 SOL) → withdraw(0.5 SOL,
github) → close. The withdraw test additionally decodes the `prereqs` PDA and asserts
the recorded GitHub username matches the argument passed in.
