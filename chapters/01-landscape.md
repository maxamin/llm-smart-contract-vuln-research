# The Landscape: What LLMs Can and Cannot Find

Smart contracts are a good target for language models. The code is small, public and high-value, and most catastrophic bugs are **logic bugs** that pattern-based static analysers miss. They are also an unforgiving one: a model that reports twenty plausible but false "reentrancy" findings wastes an auditor's day, and a model that misses the one rounding bug costs millions.

## Where the money is lost

Public loss datasets (DeFiHackLabs, Rekt, the academic Web3Bugs study) agree that the largest categories are not the textbook SWC entries. They are:

1. **Price/oracle manipulation**, usually via flash loans and spot-price reads.
2. **Business-logic errors**: wrong accounting order, missing state updates, incorrect reward maths.
3. **Access control**: unprotected initialisers, missing modifiers, misconfigured roles.
4. **Off-chain and key compromise** (outside the contract, covered in the web3-supply-chain-security book).
5. **Rounding and precision**: share inflation, `mulDiv` direction errors, dust accumulation.

Zhang et al. (ICSE 2023) found that most exploitable bugs in their Code4rena-derived dataset were **not detectable** by existing automated tools. That gap is what LLM-based research is trying to close.

## What models are good at

- Reading intent from names, NatSpec and docs, then spotting code that contradicts it.
- Recognising **known patterns in new disguises** (a reentrancy through an ERC-777 hook or an ERC-721 `onERC721Received` callback).
- Proposing **invariants and attack hypotheses** for a human or fuzzer to test.
- Explaining unfamiliar code quickly (inline assembly, unusual proxies).

## What models are bad at

- **Arithmetic over many steps**: they will confidently mis-evaluate a fixed-point formula.
- **Cross-contract reasoning** beyond the context window, or when interfaces hide behaviour.
- **Knowing what is *not* reachable**: they flag guarded code as vulnerable.
- **Calibration**: "high severity" in model output carries little information.

## The core principle of this book

> A model's claim is a **hypothesis**. A finding exists only when a tool (test, fuzzer, symbolic engine, fork replay) produces evidence.

Every chapter builds pipelines on this principle: GPTScan confirms with static analysis, PropertyGPT confirms with formal verification, and our agent (Chapter 7) confirms with Foundry proofs of concept.

## Matching each bug class to its checker

The principle above only works if every category has a checker that can turn a hypothesis into evidence. The table shows which tool decides for which kind of claim.

| Category | Typical model hypothesis | Checker that decides |
|---|---|---|
| Oracle manipulation | "price read from spot reserves" | Foundry fork test that moves reserves inside one transaction, then calls the victim |
| Business logic | "rewards updated after balance change" | unit test comparing expected vs actual reward; invariant fuzzing |
| Access control | "`initialize` callable by anyone" | Slither detectors (`unprotected-upgrade`, `missing-modifier`-style checks) plus a one-line test |
| Rounding | "`convertToShares` rounds in user's favour" | Halmos or Certora rule; fuzzed round-trip test |
| Reentrancy | "state written after external call" | Slither `reentrancy-*` detectors plus a malicious-receiver test |

Note the pattern: a model claim names a function and a mechanism. That is enough to write a test. If a claim is too vague to test ("this contract may be vulnerable to manipulation"), discard it.

## Why "logic bugs" resist both tools and models

Static analysers encode syntactic patterns. Most expensive bugs break an **intended** property that is written down only in documentation, a whitepaper or the developers' heads. Euler Finance's 2023 incident, for example, was widely reported as an accounting flaw around donations and liquidation, reachable through ordinary public functions, with losses reported at around $200 million (most funds were later returned). No syntax is wrong in such code. To find it, someone has to know what the protocol *should* do, which is where models help, and then show that the code does something else, which is where tests help.

## Worked example: hypothesis to evidence on a toy vault

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

This exercise shows the full loop on code you control. Never run it against deployed contracts.

**Step 1 – a vulnerable contract.** Create a Foundry project (`forge init toy && cd toy`) and add a vault with a reentrancy flaw:

```solidity
// src/Bank.sol  -- deliberately vulnerable teaching example
pragma solidity ^0.8.24;
contract Bank {
    mapping(address => uint256) public balance;
    function deposit() external payable { balance[msg.sender] += msg.value; }
    function withdraw() external {
        uint256 amt = balance[msg.sender];
        (bool ok,) = msg.sender.call{value: amt}("");   // external call first
        require(ok);
        balance[msg.sender] = 0;                          // state update after
    }
}
```

**Step 2 – ask the model** for findings in the reentrancy category only, with line numbers, and "say none" if there are none. A typical answer names `withdraw`, the call on the external-call line, and the late state write.

**Step 3 – mechanical confirmation with Slither.**

```bash
slither src/Bank.sol --detect reentrancy-eth
```

Expected output includes `Reentrancy in Bank.withdraw()` with the external call and the state variable written after it.

**Step 4 – executable evidence.** Write a test with a receiver that re-enters once and records the extra balance. It proves the bug without building a reusable drainer:

```solidity
// test/Bank.t.sol
import "forge-std/Test.sol"; import "../src/Bank.sol";
contract Reenter {
    Bank b; uint256 public hits;
    constructor(Bank _b) { b = _b; }
    function go() external payable { b.deposit{value: msg.value}(); b.withdraw(); }
    receive() external payable { if (hits++ == 0) b.withdraw(); }
}
contract BankTest is Test {
    function test_reentrancy_double_withdraw() public {
        Bank b = new Bank();
        vm.deal(address(this), 2 ether);
        b.deposit{value: 1 ether}();                    // another user's funds
        Reenter r = new Reenter(b);
        r.go{value: 1 ether}();
        assertEq(address(r).balance, 2 ether, "received twice its deposit");
    }
}
```

If `forge test -vv` passes, it confirms the hypothesis: the attacker contract received two ether after depositing one. Apply the fix (update state before the call, or use OpenZeppelin `ReentrancyGuard`), rerun, and the test should now fail its assertion. A finding that is not confirmed by Slither or a test stays a hypothesis.

## Common pitfalls

- **Asking "find all vulnerabilities".** Open-ended prompts yield long, generic lists. Ask per category and require line numbers.
- **Accepting severity labels from the model.** Severity depends on reachability and value at risk, which only tests and context establish.
- **Ignoring the deployment context.** A "missing access control" on a function that is never deployed, or is behind a proxy admin, may be irrelevant.
- **Benchmarking on famous incidents.** Models may remember the post-mortem. See Chapter 5.
- **Trusting arithmetic in prose.** Recompute every formula the model evaluates, ideally in a test with concrete numbers.

## Exercises

1. Pick five incidents from DeFiHackLabs. Classify each into the categories above. Would Slither have flagged any of them?
2. Give a model one vulnerable contract with no hints, then with the docs. Compare the findings.
3. Take a clean, audited contract and ask a model for vulnerabilities. Count the false positives.

## Further reading

- Zhang et al., "Demystifying Exploitable Bugs in Smart Contracts", ICSE 2023 (Web3Bugs): <https://github.com/ZhangZhuoSJTU/Web3Bugs>
- DeFiHackLabs: <https://github.com/SunWeb3Sec/DeFiHackLabs>
- Rekt leaderboard: <https://rekt.news/leaderboard/>
- Solodit (aggregated audit findings): <https://solodit.cyfrin.io>
