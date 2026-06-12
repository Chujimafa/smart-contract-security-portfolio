# Smart Contract Security Portfolio

> Business Developer · Smart Contract Security Researcher. Berlin/Brussels based. Specializing in DeFi logic vulnerabilities, formal verification, and competitive auditing.

**[LinkedIn](https://www.linkedin.com/in/chujun-fan-9354a0201)** · **[GitHub](https://github.com/Chujimafa)**

---

## Competitive Audit Results

### CodeHawks

| Project               | Protocol Type               | High  | Medium |  Low  | Report                                                                              |                                    Repo                                     |
| :-------------------- | :-------------------------- | :---: | :----: | :---: | :---------------------------------------------------------------------------------- | :-------------------------------------------------------------------------: |
| **Brivault**          | Yield Aggregator (ERC-4626) |   3   |   1    |   —   | [Findings](./competitive-audit/CodeHawks/BriVault_Results_and_Findings.md)          |     [🔗](https://github.com/Chujimafa/2025-11-brivault-security-review)      |
| **Beatland Festival** | ERC-1155                    |   1   |   2    |   —   | [Findings](./competitive-audit/CodeHawks/Beatland_Festival_Results_and_Findings.md) |  [🔗](https://github.com/Chujimafa/2025-beatland-festival-security-review)   |
| **MultiSig Timelock** | Governance                  |   1   |   1    |   1   | [Findings](./competitive-audit/CodeHawks/Multisig_TimeLock_Results_and_Findings.md) | [🔗](https://github.com/Chujimafa/2025-12-multisig-timelock-security-review) |

### Radcipher Audit Arena

**Rank:** 11 · [Leaderboard](https://radcipher.com/competition)

### Public Contests — Immunefi · Sherlock · Cantina · Code4rena

> Valid findings confirmed across multiple platforms. Reports pending official disclosure — links will be added upon publication.

---

## Solo Audit Reports

A collection of structured security reviews and PoC exploits for DeFi protocols.

**[View All Reports →](./reports/)**

---

## Research & Publications

### Damn Vulnerable DeFi — Full Exploit Series

All 18 challenges solved with executable Foundry PoCs.

**[GitHub: exploit solutions](https://github.com/Chujimafa/damn-vulnerable-defi-v4-solutions)**

*Published walkthroughs on Medium:*

- [Challenges 1–18 Full Breakdown](https://medium.com/coinsbench/damn-vulnerable-defi-v4-walkthrough-challenges-1-18-breakdown-6316d34a8492)
- [Challenge 2 — Naive Receiver](https://medium.com/@maggie.chujifan/damn-vulnerable-defi-challenges-2-naive-receiver-walkthrough-723da175878a)
- [Challenge 17 — Curvy Puppet](https://medium.com/@maggie.chujifan/damn-vulnerable-defi-challenges-17-curvy-puppet-walkthrough-0147fef8b737)

---

## Technical Stack

| Category                | Tools                                 |
| :---------------------- | :------------------------------------ |
| **Languages**           | Solidity, JavaScript                  |
| **Frameworks**          | Foundry, Hardhat                      |
| **Fuzzing**             | Foundry (stateful/stateless), Echidna |
| **Formal Verification** | Halmos, Certora (CVL)                 |

### Methodology

1. **Manual Review** — Business logic, access control, integration risks (flash loans, oracle manipulation)
2. **Property-based Testing** — Invariant definition, fuzzing, symbolic execution
3. **PoC Development** — Executable Foundry exploits for every confirmed finding

---


