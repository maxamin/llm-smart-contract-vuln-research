# Beyond the EVM: Solana, Move and Others

Most LLM security research targets Solidity because that is where the training data and datasets are. Other ecosystems are **less covered by tools and models**, which makes them a research opportunity and a risk.

## Solana (Rust, Anchor)

Solana programs are stateless. Every account a program touches is passed in by the caller, so the dominant bug classes are about **validating accounts**:

| Bug | What to check |
|---|---|
| Missing signer check | `is_signer` / Anchor `Signer<'info>` on authorities |
| Missing owner check | account owned by the expected program |
| Account type confusion | discriminators; Anchor `Account<'info, T>` handles this |
| PDA seed / bump issues | canonical bump, seeds include all identifying data |
| Arbitrary CPI | invoked program ID validated |
| Arithmetic | release builds wrap unless `overflow-checks = true` |
| Closing accounts | lamports drained *and* data zeroed / discriminator set |

Prompt models with the **accounts struct** separately from the instruction body: "for each account, state which constraint ensures it is the intended account." Anchor constraints (`has_one`, `seeds`, `constraint =`) are explicit enough that models check them well. Native (non-Anchor) programs are harder and under-studied.

Tools: Anchor's built-in checks, Trident (fuzzing), and Sec3 X-ray (commercial). Coverage of open static analysers is thinner than for the EVM.

## Move (Sui, Aptos)

Move's resource types prevent many EVM bug classes (no duplication or implicit discard of assets), and the **Move Prover** offers formal verification in the language toolchain. Remaining issues:

- **Capability leaks:** returning or storing an admin capability where others can get it.
- **Object ownership confusion (Sui):** shared vs owned objects, and who can pass which object.
- **Arithmetic and economic logic:** the language does not save you from wrong formulas.
- **Generic type parameter misuse:** accepting `Coin<T>` for any `T` when a specific type was intended.

Models have seen little Move, so retrieval with the Move Book and framework sources in context helps a lot.

## Other targets worth a look

- **CosmWasm** (Rust): message handling, reply handling, and IBC callbacks.
- **Cairo / Starknet**: felt arithmetic (field elements, not integers) is a frequent source of confusion.
- **Bitcoin scripts and L2s** (BitVM-style constructions): new and very little tooling.
- **TON (FunC/Tact)**: asynchronous message model; bounce handling bugs.

## Checkers by ecosystem

The house principle (the model proposes, a tool decides) holds everywhere, but the available tools differ.

| Ecosystem | Static / lint | Testing and fuzzing | Formal |
|---|---|---|---|
| Solana (Anchor, native) | Anchor constraints, `cargo clippy`, community linters | Trident, `solana-program-test`, LiteSVM, Bankrun-style test harnesses | limited, research-level |
| Sui / Aptos (Move) | Move compiler and linters (`sui move build --lint`) | Move unit tests (`#[test]`) | Move Prover (`aptos move prove`) |
| CosmWasm | `cargo clippy` | `cw-multi-test` | limited |
| Starknet (Cairo) | compiler warnings | Starknet Foundry (`snforge`) with fuzzing | limited |

Where formal tools are thin, executable tests carry more weight. A model claim on Solana should end in a failing `solana-program-test` or Trident case, not in a paragraph.

## Worked example: account validation review for an Anchor program

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

Use the `sealevel-attacks` repository, which pairs insecure, secure and recommended versions of common Solana mistakes.

**Step 1 – get the code.**

```bash
git clone https://github.com/coral-xyz/sealevel-attacks
ls sealevel-attacks/programs
# e.g. 0-signer-authorization, 1-account-data-matching, 2-owner-checks, ...
```

**Step 2 – extract the accounts struct.** For each `insecure` program, copy the `#[derive(Accounts)]` struct and the instruction body into separate prompt sections.

**Step 3 – structured prompt.** Ask for a table with one row per account: name, type (`AccountInfo`, `Account<T>`, `Signer`, `Program`), constraints present, and "what stops an attacker from passing a different account here?". Require "nothing" as an allowed answer.

**Step 4 – check against the secure version.** The repository's `secure` and `recommended` variants are the ground truth. Each account the model flags should correspond to a constraint the secure version adds. For example, the signer-authorization case is fixed by a signer check (`Signer<'info>` or `is_signer`).

**Step 5 – executable evidence.** For one flagged account, write an Anchor test (TypeScript or Rust) that calls the insecure instruction with an unsigned authority or a wrong-owner account on a local validator, and shows it succeeds. Then run the same test against the secure version and show it fails with a constraint error. Use only `solana-test-validator` or an in-process test harness, never mainnet.

**Step 6 – measure.** Count rows where the model's verdict matched the secure diff. On these small examples models do well; on native programs that parse accounts by index, accuracy drops, which is exactly where to focus human review.

## Move-specific prompting

For Move, give the model the struct definitions with their **abilities** (`key`, `store`, `copy`, `drop`) and ask: "which functions can produce or transfer a value of this type, and who can call them?" Abilities are mechanical facts the compiler enforces, so a model reasoning from them is reasoning from ground truth. Capability leaks show up as public functions that return an admin-capability type or put it in a shared object.

## Common pitfalls

- **Applying EVM intuitions.** There is no reentrancy in the EVM sense on Solana, and no `msg.sender`; authority comes from signer accounts. Prompts must say which ecosystem rules apply.
- **Trusting Anchor to check everything.** `AccountInfo` and `UncheckedAccount` bypass Anchor's type checks; every one deserves a manual question.
- **Forgetting release-mode arithmetic.** Rust integer overflow wraps in release builds unless `overflow-checks = true` is set in `Cargo.toml`.
- **Assuming the Move Prover proves economics.** It proves the specs you write; a missing spec proves nothing.
- **Thin training data.** Models invent APIs for Move, Cairo and TON. Put the framework source or docs in context and compile everything.

## Exercises

1. Write a Solana account-validation checklist prompt and test it on the Sealevel Attacks examples.
2. Ask a model to explain why a given Move function cannot double-spend, and verify with the Move Prover.
3. Port one EVM taxonomy category to Cairo and describe what changes.

## Further reading

- Coral/Sealevel attacks examples: <https://github.com/coral-xyz/sealevel-attacks>
- Anchor book: <https://www.anchor-lang.com>
- Move Prover: <https://aptos.dev/en/build/smart-contracts/prover>
- Sui Move book: <https://move-book.com>
- CosmWasm security notes (Oak Security): <https://github.com/oak-security/cosmwasm-security-dojo>
