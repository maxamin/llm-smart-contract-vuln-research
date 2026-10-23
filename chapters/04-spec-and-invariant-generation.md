# Specification and Invariant Generation

The highest-value use of models in contract security may not be *finding* bugs at all, but **writing the properties** that fuzzers and provers check. Properties are what auditors and developers skip under time pressure.

## PropertyGPT

PropertyGPT (Liu et al., NDSS 2025) retrieves similar, human-written properties from a database of existing formal specs (e.g. Certora rules from past audits), adapts them to the target, and **compiles and verifies** them with a formal checker in a feedback loop. Two ideas to copy:

1. **Retrieval over reference specs** beats asking from scratch. The model adapts a proven property shape instead of inventing one.
2. **Compiler and prover feedback** fixes syntax and catches vacuous properties.

## Property shapes that transfer between protocols

```
# conservation
sum(balances) == totalSupply

# solvency
asset.balanceOf(vault) >= vault.totalAssets() - pendingFees

# no free lunch (round trip)
redeem(deposit(x)) <= x

# monotonic
after(f) => exchangeRate >= before.exchangeRate      (if no losses)

# access
any change of owner => msg.sender == old owner

# liveness-ish (bounded)
healthy position => liquidate() reverts
```

## Formats and targets

| Target | Property format |
|---|---|
| Foundry fuzz/invariant | Solidity `invariant_*` functions |
| Echidna / Medusa | `echidna_*` boolean functions or assertions |
| Halmos | Solidity tests with symbolic inputs (`svm.createUint256`) |
| Certora Prover | CVL rules and invariants |
| Scribble | inline annotations compiled to runtime checks |

Generating the *same* property for several engines is a useful cross-check. If a fuzzer finds a counterexample the prover said was impossible, one of the specs is wrong.

## Detecting bad properties

- **Vacuity:** the precondition is never satisfiable, so the property is trivially true. Provers can check this (Certora's sanity rules). For fuzzers, measure how often the precondition holds.
- **Tautology:** `x == x`-style properties a model emits under pressure. Mutation testing catches these: they kill nothing.
- **Over-strong properties:** fail on legitimate behaviour (e.g. ignoring fees). These need human judgement; a model-generated explanation of the counterexample speeds that up.

## A loop you can build in a weekend

```
contract + docs ──► LLM drafts N properties (Foundry syntax)
              ──► forge build   (fix loop)
              ──► forge test --fuzz-runs 10000
                     fails → LLM explains counterexample → human: bug or bad property?
                     passes → mutation testing (gambit) → keep if kills ≥ 1 mutant
```

## Where properties come from

A model with no source material invents generic properties. Better sources, in rough order of value:

1. **The standard the code implements.** ERC-20, ERC-4626 and ERC-721 specify behaviour precisely enough to translate sentences into properties. Libraries such as a16z's `erc4626-tests` already encode many of them.
2. **Protocol documentation and whitepaper formulas.** "Interest accrues continuously at rate r" becomes a monotonicity property on the index.
3. **Past audit findings for similar protocols.** Each high-severity finding is a property someone violated. Solodit is the largest public source.
4. **Developer tests.** Existing unit tests encode expected behaviour for specific values; a model can generalise them into fuzz properties.
5. **Code comments and NatSpec.** Useful, but they can be wrong or stale, so treat them as weaker evidence.

## Worked example: one property, three engines

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

The aim is to express the round-trip property for a vault in Foundry, Halmos and Certora syntax and see whether the engines agree.

**Step 1 – Foundry fuzz test.**

```solidity
function testFuzz_roundTrip(uint256 assets) public {
    assets = bound(assets, 1, 1e30);
    asset.mint(address(this), assets);
    asset.approve(address(vault), assets);
    uint256 shares = vault.deposit(assets, address(this));
    uint256 back = vault.redeem(shares, address(this), address(this));
    assertLe(back, assets, "no free lunch");
}
```

Run with `forge test --match-test roundTrip --fuzz-runs 10000`.

**Step 2 – Halmos symbolic test.** Halmos runs functions prefixed `check_` with symbolic arguments:

```solidity
function check_roundTrip(uint256 assets) public {
    vm.assume(assets > 0 && assets < 1e30);
    asset.mint(address(this), assets);
    asset.approve(address(vault), assets);
    uint256 shares = vault.deposit(assets, address(this));
    uint256 back = vault.redeem(shares, address(this), address(this));
    assert(back <= assets);
}
```

Run `halmos --function check_roundTrip`. Halmos reports `[PASS]` or a counterexample with concrete values. Note that this checks a single deposit from an empty vault; add a symbolic prior deposit and a symbolic donation to cover more states.

**Step 3 – Certora rule (sketch).**

```text
rule roundTrip(uint256 assets) {
    env e;
    require assets > 0;
    uint256 shares = deposit(e, assets, e.msg.sender);
    uint256 back = redeem(e, shares, e.msg.sender, e.msg.sender);
    assert back <= assets;
}
```

Certora starts from an arbitrary state rather than the constructor, so it explores states that the fuzzer and Halmos setup may not reach. That often surfaces counterexamples that depend on unusual `totalSupply`/`totalAssets` ratios; decide whether those states are reachable before calling it a bug.

**Step 4 – compare.** If all three agree the property holds, keep it. If one finds a counterexample, replay it as a concrete Foundry test. Disagreement is information: either a setup difference, an unreachable state, or a real bug one engine missed.

## Common pitfalls

- **Properties that only hold from the constructor state.** Fuzzers start from `setUp()`, provers from any state. A property may pass one and fail the other for legitimate reasons. Document the assumed state.
- **`vm.assume` that filters out almost everything.** Over-tight assumptions make Halmos and fuzzers vacuous. Count how many runs were rejected.
- **Ignoring fees and yield.** Strict round-trip equality fails on any fee-charging vault. Encode the fee in the property.
- **Trusting the model's translation of CVL.** CVL has its own semantics (`require`, `env`, ghost and hook rules). Compile with the Certora tooling and read the sanity results.
- **No mutation check.** A suite that kills no mutants is decoration, however many properties it contains.

## Exercises

1. Build a small database of 30 invariants from public audit repos. Retrieve the top 3 for a new protocol and adapt them.
2. Write the same solvency property for Foundry and Halmos. Do they agree on a deliberately buggy vault?
3. Measure the vacuity rate of 20 model-generated properties.

## Further reading

- Liu et al., "PropertyGPT: LLM-driven Formal Verification of Smart Contracts through Retrieval-Augmented Property Generation", NDSS 2025, arXiv:2405.02580
- Certora documentation: <https://docs.certora.com>
- Gambit mutation tester: <https://github.com/Certora/gambit>
- Halmos: <https://github.com/a16z/halmos>
- Scribble: <https://github.com/Consensys/scribble>
- See **neurosymbolic-contract-verification** in this wiki.
