# Glossary

- **Invariant:** a property that must hold in every reachable state.
- **Read-only reentrancy:** reading inconsistent state through a view function during a callback.
- **Flash loan:** an uncollateralised loan repaid in the same transaction; amplifies manipulation.
- **TWAP:** time-weighted average price.
- **ERC-4626:** tokenised vault standard; known for share-inflation edge cases.
- **PDA:** Program Derived Address (Solana).
- **CPI:** Cross-Program Invocation (Solana).
- **Capability:** a Move object/resource granting a permission.
- **Vacuous property:** a property that is true only because its precondition never holds.
- **PoC:** proof of concept, here an executable test that demonstrates the bug.
- **Abilities (Move):** `key`, `store`, `copy`, `drop` markers that the compiler uses to restrict what can happen to a type.
- **Evidence gate:** a tool that accepts a finding only with executable evidence, such as a passing PoC test.
- **Facts block:** analyser-derived facts (reads, writes, calls, modifiers) injected into a prompt so the model does not have to derive them.
- **Fork test:** a test that runs against a local copy of chain state pinned to a block number.
- **Memorisation test:** re-running an evaluation on renamed or restructured code to see whether performance depends on recalling known examples.
- **Refutation memory:** a store of hypotheses already disproved, fed back to avoid repeated work.
- **Round-trip property:** depositing then withdrawing never returns more than was put in.
- **Scenario / property split:** GPTScan's approach of first matching a usage scenario, then checking a vulnerable property.
