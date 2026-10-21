# Video script — pre-req-vault architecture walkthrough (≤ 3 minutes)

Tone: speak to a new engineer joining the project, diagram on screen
(`docs/architecture.svg`), point at the relevant boxes per section.
~430 words ≈ 2:50 at a relaxed pace. Practice once, trim if over.

---

**[0:00 → 0:25] What this program is**

"Hi — this is the Turbin3 vault prerequisite. It's a small Anchor program that
gives every wallet one private vault for SOL. What makes it interesting is the
withdraw instruction: after sending your money back, it cross-program-invokes
Turbin3's registration program on devnet and records my GitHub handle on
chain. Everything I'm pointing at is in the architecture diagram."

**[0:25 → 1:05] The accounts — left and middle of the diagram**

"Two PDAs belong to the vault program and everything hangs off the user's
keypair. First, VaultState, derived from 'state' plus the user key: it just
stores two bump numbers — one for itself, one for the vault PDA. The vault PDA
is derived from 'vault' plus the VaultState address — I'll come back to why
it's indirect. The vault holds the actual SOL. It has no data and no private
key; the program signs for it with its seeds, plus the bump it stored earlier."

**[1:05 → 1:50] The four instructions**

"Initialize creates VaultState, paid by the user, and stores those bumps.
Deposit does a System-program transfer from the user into the vault — Solana
creates the vault account on the first deposit. Withdraw is the interesting
one: the program transfers back a chosen amount, signing for the vault with its
seeds, and then performs the CPI. Close drains whatever remains and closes
VaultState, returning rent. So the lifecycle is init → deposit → withdraw, and
close ends the round trip."

**[1:50 → 2:35] The CPI — the thick amber arrow**

"The CPI is the heart of the challenge. When our program calls Turbin3's
initialize, three accounts matter. The application account, derived from
'prereqs' plus the user — and note the derivation is explicitly pinned to the
registration program's ID, so nobody can point it at a fake program. The
application program account itself, which Anchor validates is the real TRBZ
program. And the user — here's the neat part: the user's original signature on
the top-level transaction propagates through the CPI automatically, so the
registration program sees a properly signed owner. Turbin3's program then
creates that prereqs account once — only once per wallet — and writes the
GitHub username I passed as an argument."

**[2:35 → 2:50] Proof it works**

"I deployed my own build — the template's address belongs to Turbin3 — and ran
the test suite on devnet: all four legs pass, and the withdraw test also reads
the prereqs account back and asserts the recorded handle equals the argument.
You can see the account and the four transactions in the README."

**[2:50 → end] Close**

"That's the full flow: one wallet, two program-owned PDAs, one external CPI.
Thanks for watching."

---

## Recording checklist

- [ ] Upload to YouTube, visibility **unlisted or public** (not private)
- [ ] Add **captions** (auto-generate, then fix the program names if mangled)
- [ ] Keep ≤ 3:00 — if long, tighten section [1:05 → 1:50] first
- [ ] Paste the link into `README.md` (line marked "video: _add your YouTube link here_")

## If someone asks the boring-but-real detail (freeze-risk answers)

- Why is `vault` derived from `vault_state` rather than directly from the user?
  → So `VaultState` holds its child's bump and both bumps are fetchable from a
  single account; it's a template convention of this program.
- Why does `close` also refund? → It doubles as the final withdrawal: drains the
  vault first, then `close = user` on `VaultState` returns its rent.
- Why hardcode amounts in tests? → The devnet run only needs the flows to hold;
  1 SOL deposit / 0.5 SOL withdraw from the template are kept as-is.
- Where is the `github` argument encoded? → Anchor appends it to the instruction
  data after the 8-byte discriminator; the registration program stores it in
  `ApplicationAccount.github` (a String, 4-byte length prefix + UTF-8 bytes).
