# Datasets and Benchmarks

You cannot improve what you cannot measure, and contract-security benchmarks have quirks that make results misleading.

## Main datasets

| Dataset | Content | Caveats |
|---|---|---|
| SmartBugs Curated | ~140 small contracts, labelled by SWC-like category | old Solidity, toy-sized, heavily in training data |
| SmartBugs Wild | ~47k real contracts, unlabelled | needs your own labels |
| SolidiFI | bugs injected into real contracts | injected bugs look injected |
| Web3Bugs | exploitable bugs from Code4rena contests | realistic, but the reports are public |
| DeFiHackLabs | Foundry reproductions of real exploits | ground truth is exact; excellent for exploit synthesis |
| Solodit | tens of thousands of audit findings | great for retrieval; noisy severities |
| DAppSCAN | audit-report-derived weaknesses in DApp projects | broad, mixed quality |

## Contamination is the main threat

Everything above is public and likely in model training data. A model "finding" a well-known bug may just be recalling the report. Mitigations:

1. **Temporal split:** evaluate only on contests or incidents after the model's training cutoff.
2. **Renaming and restructuring:** rename identifiers, reorder functions, and strip comments, then check whether performance drops. A large drop suggests memorisation.
3. **Private holdout:** audit firms with unpublished findings can build clean sets. Ask for partnerships.
4. **Synthetic-but-realistic bugs:** mutate audited code with domain-aware mutations (swap rounding direction, drop a staleness check) rather than generic operators.

## Metrics that matter

- **Precision at k** (k = how many findings a human will actually read, often 10).
- **Recall on high/critical findings** separately from low/informational ones.
- **Location accuracy:** correct function and lines, not just "this contract has reentrancy".
- **Cost per true positive** (tokens, wall time).
- **Explanation quality:** does the explanation identify the *real* root cause? Needs human grading with a rubric.

## Build your own living benchmark

```
benchmark/
  cases/
    2026-03-protocolX-H01/
      src/            # code at audit commit
      finding.md      # ground truth: root cause, lines, severity
      poc.t.sol       # Foundry test that fails on vuln, passes on fix
      meta.json       # date, source, chain, categories
  runner/
    run.py            # tool → findings.json
    score.py          # match findings to ground truth by function + category
```

The `poc.t.sol` makes the ground truth *executable*, and lets you score tools that output PoCs.

## Choosing a dataset for the question you are asking

Different questions need different ground truth. Using SmartBugs Curated to claim "our tool finds DeFi logic bugs" is a category error.

| Question | Suitable data | Why |
|---|---|---|
| Does my detector recognise classic patterns? | SmartBugs Curated, SolidiFI | labelled, small, quick to run |
| Does it find bugs auditors actually report? | Web3Bugs, post-cutoff contest findings | realistic, human-verified |
| Can it reproduce a real exploit end to end? | DeFiHackLabs | executable PoCs on pinned forks |
| What is its false-positive rate on real code? | SmartBugs Wild sample, plus manual labels | real distribution, mostly clean code |
| Is it memorising? | any set, before and after renaming | measures the drop under perturbation |

For most LLM work, combine one realistic set (to measure recall on hard bugs) with one large clean set (to measure false positives per thousand lines). A tool that finds everything and flags everything is not useful.

## Worked example: a renaming test for memorisation

Not executed in this book's CI; the commands, code and expected outputs below are illustrative. Verify locally before relying on them.

The question: does a model's performance on a public benchmark depend on recognising the exact code?

**Step 1 – pick cases.** Take 30 cases from SmartBugs Curated (for example the `reentrancy` and `access_control` folders) and 30 post-cutoff contest findings you have labelled.

**Step 2 – perturb.** Rename identifiers and strip comments while keeping semantics. For Solidity, a simple, safe approach is to rename contract, function and state variable names consistently with a mapping, then recompile to prove nothing broke:

```bash
solc --version
for f in cases/*/src/*.sol; do
  python3 rename.py "$f" > "perturbed/$(basename $f)"   # your consistent-renaming script
  solc --bin "perturbed/$(basename $f)" > /dev/null || echo "BROKE: $f"
done
```

A robust `rename.py` uses a parser (for example the `solidity-parser` npm package or Slither's AST) rather than regular expressions, so it does not rename inside strings or collide with keywords.

**Step 3 – run the tool** on original and perturbed sets with fixed prompts, temperature and model version.

**Step 4 – score** with the same matching rule (function plus category). Report recall on each set:

| Set | Original recall | Perturbed recall | Drop |
|---|---|---|---|
| SmartBugs Curated (pre-cutoff) | measured | measured | ? |
| Post-cutoff findings | measured | measured | ? |

**Step 5 – interpret.** A large drop on the old set and a small drop on the new set suggests memorisation of the old set. A small drop on both suggests the tool reasons from code. Repeat runs to estimate variance before concluding anything.

## Matching rules in practice

Matching tool findings to ground truth decides your numbers. A practical rule has three levels:

1. **Exact:** same function and same category, and the tool's lines overlap the ground-truth lines.
2. **Partial:** same function, different but related category (for example "reentrancy" for a read-only reentrancy bug).
3. **Miss:** anything else, including the right category in the wrong function.

Report exact and exact-plus-partial separately. If you let a model judge matches, give it the ground truth and the finding and ask a yes/no question with a cited reason, then audit a random sample by hand. The **synthetic-vulnerability-datasets** book covers scoring in more depth.

## Common pitfalls

- **Scoring at contract level.** "This contract has a bug" is almost always true of benchmark contracts. Score at function and line level.
- **Old compiler versions.** Many curated cases need Solidity 0.4.x. Tools that fail to compile them silently score zero recall. Track compile failures separately.
- **Treating severity labels as consistent.** Contest severities depend on judges and rules. Normalise before comparing.
- **No clean set.** Without code that has no known bugs, you cannot measure false positives.
- **Using one run.** LLM outputs vary. Run at least three times and report the spread.

## Exercises

1. Run any LLM tool on SmartBugs Curated before and after identifier renaming. Report the delta.
2. Add five post-cutoff contest findings to a living benchmark with PoCs.
3. Define a matching rule between tool findings and ground truth that is neither too strict nor too lenient, and justify it.

## Further reading

- SmartBugs: <https://github.com/smartbugs/smartbugs>; Durieux et al., ICSE 2020
- SolidiFI: Ghaleb & Pattabiraman, ISSTA 2020
- Web3Bugs: <https://github.com/ZhangZhuoSJTU/Web3Bugs>
- DeFiHackLabs: <https://github.com/SunWeb3Sec/DeFiHackLabs>
- Solodit: <https://solodit.cyfrin.io>
