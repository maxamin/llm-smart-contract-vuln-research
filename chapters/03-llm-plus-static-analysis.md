# LLM + Static Analysis: The GPTScan Pattern

GPTScan (Sun et al., ICSE 2024) set out a pattern worth copying: **use the model to *recognise* a scenario, then use static analysis to *confirm* the key properties.** It reported much better precision than letting the model decide alone.

## The pattern in three steps

1. **Scenario + property description.** Each bug type is described in two parts, for example:
   - *Scenario:* "the function performs a token swap that relies on a price from an AMM pair."
   - *Property:* "the price is computed from current reserves and no slippage bound is enforced."
2. **Model matching.** For each function, ask a yes/no question: does it match the scenario? Then: does it have the property? Filtering by scenario first cuts cost and noise.
3. **Static confirmation.** The model also names the *key variables and statements* (e.g. "the reserve read is `getReserves()` at L88; the amountOutMin is `0`"). A static checker (data flow, call order) verifies those claims. Unconfirmed matches are dropped.

## Building it with Slither

Slither exposes an IR (SlithIR) and a Python API, which makes it a natural confirmation backend:

```python
from slither import Slither
sl = Slither("contracts/")
for c in sl.contracts:
    for f in c.functions:
        ext_calls = [n for n in f.nodes if n.high_level_calls or n.low_level_calls]
        writes_after = [n for n in f.nodes
                        if n.state_variables_written and any(e.node_id < n.node_id for e in ext_calls)]
        # feed f.source_mapping + facts to the LLM; then verify the LLM's claimed
        # variable names against f.state_variables_written etc.
```

Feed the model **facts Slither already knows** (state variables read and written, modifiers, external calls) alongside the source. Models are much more reliable when they reason over facts than when they derive them.

## Confirmation checks you can automate

| Model claim | Mechanical check |
|---|---|
| "state written after external call" | node ordering in CFG |
| "no access control" | modifiers + `msg.sender` comparisons in function or callees |
| "price from spot reserves" | data flow from `getReserves`/`balanceOf(pair)` into price variable |
| "user-controlled loop bound" | taint from parameters or storage arrays with public push |
| "rounds down" | Solidity integer division, or `mulDiv` without `Up` variant |

## Cost control

- Pre-filter functions by cheap features (has external call, writes storage, is public).
- Cache answers per function hash, since audits re-run often.
- Use a small model for the scenario filter and a large one only for properties.

## Why the split works

The model and the analyser fail in opposite ways. Slither is precise about facts (this node writes that variable; this function has no modifier) but blind to intent (whether the write *should* happen first). A model is good at intent and bad at facts: it misreads which branch a `require` guards and hallucinates helper functions. Asking the model only about intent, and asking the analyser only about facts, plays each to its strength. The GPTScan authors reported that the static confirmation step removed a large share of the model's false positives while keeping most true ones; treat the exact figures in the paper as specific to its dataset.

## Worked example: oracle staleness with model matching and Slither confirmation

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

The target bug: a contract reads Chainlink's `latestRoundData()` and does not check `updatedAt` for staleness.

**Step 1 – cheap pre-filter with Slither.** Find functions that call `latestRoundData` at all:

```python
# find_oracle_reads.py
from slither import Slither
sl = Slither(".")   # Foundry/Hardhat project root, compiled via crytic-compile
cands = []
for c in sl.contracts_derived:
    for f in c.functions_and_modifiers:
        for _, call in f.all_high_level_calls():
            # older Slither: call is a Function; newer: a HighLevelCall IR
            fn = getattr(call, "function", call)
            if getattr(fn, "name", None) == "latestRoundData":
                cands.append((c.name, f.full_name, f.source_mapping.lines))
print(*cands, sep="\n")
```

The tuple layout returned by `all_high_level_calls()` has changed across Slither versions; check it against your installed version and adjust the indexing.

**Step 2 – scenario and property questions.** For each candidate, send the function source and the facts block:

```text
FACTS (from Slither):
- function: PriceFeedConsumer.getPrice()
- calls: AggregatorV3Interface.latestRoundData()
- local variables assigned from the call: answer, updatedAt
- require/if conditions in function: [answer > 0]
QUESTION 1: Does this function use the oracle answer to price assets? yes/no
QUESTION 2: Is updatedAt compared against block.timestamp with a threshold? yes/no.
Name the exact line if yes.
```

**Step 3 – confirm.** If the model says "no staleness check", verify that no condition in the function (or its callers within the same contract) reads `updatedAt`. In Slither, check whether `updatedAt` appears in the variables read by any node that has a condition (`node.contains_require_or_assert()` or `node.contains_if()`).

**Step 4 – evidence.** Write a Foundry test with a mock aggregator whose `updatedAt` is days old and show the consumer accepts the price.

**Step 5 – score.** On a labelled set of contracts, record model-only and model+confirmation precision. The difference is the value of the confirmation step on your data.

## Facts block design

The facts you inject should be short, structured and complete for the question. Good facts for most categories:

- function visibility, modifiers and whether it is `payable`;
- state variables read and written, in node order;
- external and low-level calls with their target expressions;
- `require`/`assert`/`if` conditions as source text;
- inherited functions that override or are overridden.

Avoid dumping whole SlithIR. Models handle a 20-line facts block well and a 400-line IR listing badly.

## Common pitfalls

- **Letting the model derive facts.** If the prompt does not state which variables are written, the model will guess. Inject facts from the analyser.
- **Confirming only the model's wording.** The model says "`amountOutMin` is 0", but the value comes from a parameter. Confirm with data flow, not string matching.
- **Ignoring inheritance.** Modifiers and checks often live in a parent contract. Use `contracts_derived` and `all_*` helpers that traverse inheritance.
- **Version drift in Slither's API.** Pin the version in your tooling and test the helper functions you rely on.
- **Over-filtering.** A cheap pre-filter that requires an external call misses pure accounting bugs. Keep separate pre-filters per category.

## Exercises

1. Implement scenario/property matching for "unchecked oracle staleness" with a Slither confirmation step. Measure precision on 20 contracts.
2. Compare the model alone against model + confirmation on the SmartBugs curated set for reentrancy.
3. Add a "facts block" from Slither to your prompt and measure how false positives change.

## Further reading

- Sun et al., "GPTScan: Detecting Logic Vulnerabilities in Smart Contracts by Combining GPT with Program Analysis", ICSE 2024, arXiv:2308.03314
- Slither: <https://github.com/crytic/slither> (Python API docs in the wiki)
- Aderyn (Rust static analyser by Cyfrin): <https://github.com/Cyfrin/aderyn>
- Semgrep Solidity rules (Decurity): <https://github.com/Decurity/semgrep-smart-contracts>
