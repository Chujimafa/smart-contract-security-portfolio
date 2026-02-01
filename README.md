# smart-contract-security-portfolio
A collection of security audits and PoCs focused on DeFi logic, smart contract vulnerabilities, and best-practice mitigations.

## About me 

Berlin/Brussels based. Former Architect now pivoting my structural mindset into Smart Contract Auditing. Deep diving into CTFs and DeFi protocols.

[LinkedIn Profile](https://www.linkedin.com/in/chujun-fan-9354a0201)

## Technical Stack & Methodology

### Languages & Frameworks
- **Languages:** Solidity, JavaScript 
- **Frameworks:** Foundry, Hardhat
  
### Security & Testing Suite
- **Fuzzing & Invariant Testing:** 
  - **Foundry:** Advanced stateful/stateless fuzzing and system-wide invariant development.
  - **Echidna:** Property-based fuzzing to discover deep edge cases and state transitions.
- **Formal Verification (FV):** 
  - **Halmos:** Symbolic testing for formal proofs of contract properties.
  - **Certora:** Writing CVL specifications for high-assurance logic verification.


###  Methodology
1. **Manual Review:** Line-by-line analysis focusing on business logic, access control, and integration risks (e.g., flash loans, oracle manipulation).
2. **Property-based Testing:** Defining core invariants and using fuzzing/symbolic execution to attempt to break system guarantees.
3. **PoC Development:** Crafting executable Proof-of-Concepts in Foundry to demonstrate and validate all identified vulnerabilities.

## Security Research & DeFi Challenges

### [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/)

*A collection of solutions and detailed exploit PoCs for the classic DeFi security wargame.*
- [View all my exploit solutions here](https://github.com/Chujimafa/damn-vulnerable-defi-v4-solutions)
- [**Coming Soon on Medium**]: *My Journey Through Damn Vulnerable DeFi V4.*

## Audit Reports
 A collection of structured security analysis reports and PoCs for various educational DeFi protocols. 
 
 [View My Security Review Portfolio](./reports/)

## Competitive Audits

### Codehawk Performance Dashboard

**Rank at 20260201: rgb(201, 52, 29)**

![Codehawk Rank](./competitive%20Audit/CodeHawks/Mafa-stats.png)

| Project               | Protocol Type               | High  | Medium | Low / Gas | Analysis & PoC                                                                           |                                 Github Link                                 |
| :-------------------- | :-------------------------- | :---: | :----: | :-------: | :--------------------------------------------------------------------------------------- | :-------------------------------------------------------------------------: |
| **Brivault**          | Yield Aggregator (ERC-4626) |   3   |   1    |     -     | [View Findings](./competitive-audit/CodeHawks/BriVault_Results_and_Findings.md)          |     [🔗](https://github.com/Chujimafa/2025-11-brivault-security-review)      |
| **Beatland Festival** | ERC1155                     |   1   |   2    |     -     | [View Findings](./competitive-audit/CodeHawks/Beatland_Festival_Results_and_Findings.md) |  [🔗](https://github.com/Chujimafa/2025-beatland-festival-security-review)   |
| **MultiSig Timelock** | Governance                  |   1   |   1    |     1     | [View Findings](./competitive-audit/CodeHawks/brivault_results_and_findings.md)          | [🔗](https://github.com/Chujimafa/2025-12-multisig-timelock-security-review) |



## Specialized Tooling Proof-of-Concepts
A dedicated workspace for advanced security methodologies, focusing on **Fuzzing** and **Formal Verification (FV)**. 

> **Status:** Currently building out comprehensive test suites for DeFi logic. Content coming soon.


## Featured Web3 Projects
Beyond auditing, I actively build and research complex DeFi architectures and security primitives. My development work focuses on **logic integrity**, **gas optimization**, and **advanced smart contract patterns**.

> [!TIP]
> **[View My Full Project Directory & Security Specifications](./web3-project/README.md)**
> *Includes Deep Dives into: Stablecoin Engines, Cross-Chain Protocols, and Secure Governance.*