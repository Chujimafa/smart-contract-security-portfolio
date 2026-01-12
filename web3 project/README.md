# Web3 Project Directory

---

##  Featured Web3 Development Projects

### 1. Multi-Signature Wallet System with Safe SDK + Thirdweb Integration

| Key Technology                  | Verification Focus                                        | Repo Link                                                                                   |
| :------------------------------ | :-------------------------------------------------------- | :------------------------------------------------------------------------------------------ |
| **Safe SDK, Thirdweb, EIP-712** | **Multi-Tier Transaction Governance and Social Recovery** | [Multi-Signature-Wallet-System](https://github.com/Chujimafa/Multi-Signature-Wallet-System) |

**Project Focus:**
This system implements a robust multi-signature corporate treasury wallet using the Safe SDK and Thirdweb SDK. It features **multi-tier transaction approval logic** where the required signature threshold is automatically adjusted based on the transaction value (e.g., higher value requires more approvals). It also integrates a **Guardian-based social recovery system** inspired by time-locked recovery mechanisms.

---

### 2. Web3 Frontend DApp - NFT Pilot

| Key Technology                          | Verification Focus                                    | Repo Link                                                                       |
| :-------------------------------------- | :---------------------------------------------------- | :------------------------------------------------------------------------------ |
| **Next.js, Thirdweb, Moralis, Alchemy** | **NFT Management and Real-Time Blockchain Data APIs** | [Web3-Frontend-NFT-Pilot](https://github.com/Chujimafa/Web3-Frontend-NFT-Pilot) |

**Project Focus:**
A decentralized application (DApp) built with Next.js integrating several modern Web3 tools for data and interaction. It consists of two main parts: an **NFT Manager** (fetching, minting, and transferring ERC-721 tokens via Moralis/Thirdweb) and a **Token Explorer** (monitoring real-time ETH/ERC-20 balances and transaction history). 

---

### 3. DeFi Yield Dashboard

| Key Technology                     | Verification Focus                                      | Repo Link                                                                 |
| :--------------------------------- | :------------------------------------------------------ | :------------------------------------------------------------------------ |
| **Next.js, Hardhat, Thirdweb SDK** | **On-Chain Data Storage and Real-Time APY/TVL Display** | [Defi-Yield-Dashboard](https://github.com/Chujimafa/Defi-Yield-Dashboard) |

**Project Focus:**
A full-stack project providing a simple dashboard to display yield farming opportunities. The solution includes a **Hardhat smart contract** (`DeFiProtocolManager.sol`) for on-chain storage of protocol data (Name, APY, TVL), and a **Next.js frontend** integrated with Thirdweb SDK to fetch and display this data in a simple comparison view.

---

### 4. A Secure and Efficient ERC20 Airdrop Contract

| Key Technology                             | Verification Focus                                     | Repo Link                                                                     |
| :----------------------------------------- | :----------------------------------------------------- | :---------------------------------------------------------------------------- |
| **Solidity, Merkle Proof, EIP-712, ECDSA** | **Gasless Claim Mechanism and Eligibility Validation** | [Airdrop-and-Signatures](https://github.com/Chujimafa/Airdrop-and-Signatures) |

**Project Focus:**
This project implements an ERC20 airdrop contract that optimizes for security and efficiency. Key features include **Merkle Proof Validation** for eligible users and **EIP-712 Typed Data Signatures** validated via ECDSA for secure, gasless claiming (allowing the user to delegate the transaction cost). It is designed to be re-entrancy safe.

---

### 5. Decentralized Stablecoin (DSC) Smart Contracts

| Key Technology                                          | Verification Focus                                           | Repo Link                                                                                                                 |
| :------------------------------------------------------ | :----------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------ |
| **Solidity, Chainlink Oracles, Over-Collateralization** | **Algorithmic Stablecoin Minting and Liquidation Mechanics** | [Decentralized-Stablecoin-DSC-Smart-Contracts](https://github.com/Chujimafa/Decentralized-Stablecoin-DSC-Smart-Contracts) |

**Project Focus:**
A simplified MakerDAO-inspired decentralized stablecoin system (DSC) pegged to the US Dollar. The core logic (`DSCEngine.sol`) handles collateral deposits (wETH, wBTC), stablecoin minting, redemption, and incentivized liquidation. **Chainlink Price Feeds** are used to enforce a minimum **200% over-collateralization** via health factor checks.

---

### 6. Cross-Chain Rebase Token Protocol

| Key Technology                                     | Verification Focus                                           | Repo Link                                                                                           |
| :------------------------------------------------- | :----------------------------------------------------------- | :-------------------------------------------------------------------------------------------------- |
| **Chainlink CCIP, Rebase Token, Layer 2 Bridging** | **Cross-Chain Interest Rate Preservation and Token Minting** | [Cross-Chain-Rebase-Token-Protocol](https://github.com/Chujimafa/Cross-Chain-Rebase-Token-Protocol) |

**Project Focus:**
This protocol integrates **Chainlink CCIP** to facilitate seamless cross-chain bridging of a **Rebase Token**. The key feature is the preservation of each user's **personal, time-locked interest rate** when tokens are moved across chains (burned on source, minted on destination), ensuring yield consistency in a multi-chain environment.

---

### 7. UUPS Upgradeable Smart Contract Demo

| Key Technology                          | Verification Focus                                      | Repo Link                                                                                                        |
| :-------------------------------------- | :------------------------------------------------------ | :--------------------------------------------------------------------------------------------------------------- |
| **Foundry, OpenZeppelin, ERC1967Proxy** | **Secure UUPS Proxy Deployment and State Preservation** | [proxy-\_-UUPS-upgradeable-smart-contract](https://github.com/Chujimafa/proxy-_-UUPS-upgradeable-smart-contract) |

**Project Focus:**
A demonstration of the **UUPS (Universal Upgradeable Proxy Standard)** pattern using Foundry and OpenZeppelin. It showcases the full workflow: deploying the initial logic (`BoxV1`), deploying the proxy (`ERC1967Proxy`), and safely upgrading the proxy to a new implementation (`BoxV2`) while preserving the contract state.

---

### 8. PunkteSystem Smart Contract Development

| Key Technology                                    | Verification Focus                                           | Repo Link                                                                                                                                   |
| :------------------------------------------------ | :----------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| **Solidity, Hardhat, Inheritance, Custom Errors** | **Access Control, Interface Abstraction, and Extensibility** | [Developing-a-PunkteSystem-Smart-Contract-with-Hardhat](https://github.com/Chujimafa/Developing-a-PunkteSystem-Smart-Contract-with-Hardhat) |

**Project Focus:**
This project develops a secure and gas-efficient points system, demonstrating strong contract development practices. Key features include using an **immutable owner**, **custom errors** for restricted access, **interface abstraction** (`ILernPunkt`), and contract **inheritance** (`ErweitertesPunkteSystem` inherits from `PunkteSystem`) for structured extensibility.

---