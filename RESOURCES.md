# Resources

## Static analysis and tooling
- [Slither](https://github.com/crytic/slither): Solidity static analyser with Python API
- [Aderyn](https://github.com/Cyfrin/aderyn): Rust-based Solidity analyser
- [Mythril](https://github.com/Consensys/mythril): symbolic execution for EVM bytecode
- [Foundry](https://github.com/foundry-rs/foundry): testing, fuzzing, forking
- [Halmos](https://github.com/a16z/halmos): symbolic testing
- [Certora Prover](https://docs.certora.com) and [Gambit](https://github.com/Certora/gambit)

## Datasets
- [SmartBugs](https://github.com/smartbugs/smartbugs)
- [Web3Bugs](https://github.com/ZhangZhuoSJTU/Web3Bugs)
- [DeFiHackLabs](https://github.com/SunWeb3Sec/DeFiHackLabs)
- [Solodit](https://solodit.cyfrin.io)

## Papers
- GPTScan (ICSE 2024), arXiv:2308.03314
- PropertyGPT (NDSS 2025), arXiv:2405.02580
- LLM4Vuln, arXiv:2401.16185
- Demystifying Exploitable Bugs in Smart Contracts (ICSE 2023)

## Learning
- [Secureum](https://secureum.substack.com)
- [Building Secure Contracts](https://secure-contracts.com)
- [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz)
- [Ethernaut](https://ethernaut.openzeppelin.com)

## More tools
- [Medusa](https://github.com/crytic/medusa) and [Echidna](https://github.com/crytic/echidna): property-based fuzzers for Solidity
- [slither-check-upgradeability](https://github.com/crytic/slither/wiki/Upgradeability-Checks): proxy and storage-layout checks
- [OpenZeppelin Upgrades plugins](https://docs.openzeppelin.com/upgrades-plugins): storage layout validation between versions
- [Scribble](https://github.com/Consensys/scribble): runtime-checked annotations
- [a16z erc4626-tests](https://github.com/a16z/erc4626-tests): reusable ERC-4626 property tests
- [Sealevel attacks](https://github.com/coral-xyz/sealevel-attacks): paired insecure/secure Solana examples
- [Trident](https://github.com/Ackee-Blockchain/trident): Solana fuzzing framework
- [Starknet Foundry](https://github.com/foundry-rs/starknet-foundry): Cairo testing with fuzzing
- [Aptos Move Prover docs](https://aptos.dev/en/build/smart-contracts/prover)

## More datasets and papers
- SolidiFI: Ghaleb and Pattabiraman, "How Effective are Smart Contract Analysis Tools? Evaluating Smart Contract Static Analysis Tools Using Bug Injection", ISSTA 2020
- Durieux et al., "Empirical Review of Automated Analysis Tools on 47,587 Ethereum Smart Contracts", ICSE 2020 (SmartBugs)
- DAppSCAN: Zheng et al., "DAppSCAN: Building Large-Scale Datasets for Smart Contract Weaknesses in DApp Projects"

## Practice targets
- [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz)
- [Ethernaut](https://ethernaut.openzeppelin.com)

## Companion books in this wiki
- **neurosymbolic-contract-verification**: proving properties instead of fuzzing them
- **ai-static-analysis-rule-writing**: turning findings into reusable detectors
- **synthetic-vulnerability-datasets**: contamination-aware benchmarks
