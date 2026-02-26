# 🔀 Intents Swap: BCH Cross-Chain HTLC Contracts

Welcome to the **BCH Cross-Chain** smart contract repository for the Intents Swap protocol! 

This repository contains the core **CashScript** smart contracts that enable **100% trustless, atomic cross-chain swaps** between Bitcoin Cash (UTXO) and account-based chains like Solana and Movement (EVM/Move).

---

## 📖 DevRel Overview

If you are explaining this architecture to developers, the core primitive you need to focus on is the **Hashed Time-Lock Contract (HTLC)**. 

Unlike traditional centralized bridges or federated multisig wrappers (which introduce custodial risk), our cross-chain architecture relies purely on **cryptographic primitives** and **Bitcoin Script**.

### The Core Contract: `CrossChainHTLC.cash`

At the heart of this repository is an elegant, 60-line [CashScript](https://cashscript.org/) contract. It does not use mutable storage or central operators. Instead, **the covenant IS the state**. 

When a user or solver locks funds, they create a deterministic P2SH address based on four constructor parameters:

```cashscript
contract CrossChainHTLC(
    bytes20 senderPkh,     // WHO locks the funds
    bytes20 recipientPkh,  // WHO can claim the funds
    bytes32 secretHash,    // SHA-256 hash of the 32-byte secret
    int timelock           // Unix timestamp expiry
)
```

### How the Magic Works (The Atomic Swap)

The atomic swap guarantees that either **both parties get the funds on their respective chains**, or **both parties get a refund**. It works via a shared cryptographic secret.

#### 1. The `claim` Function (The Happy Path)
When the solver fulfills the user's intent on the destination chain (e.g., Solana), the user must reveal the original 32-byte `secret` to claim their funds there.

Once that `secret` is revealed on Solana, the solver takes that exact same secret and passes it to the BCH covenant:

```cashscript
function claim(pubkey recipientPk, sig recipientSig, bytes32 secret) {
    require(hash160(recipientPk) == recipientPkh); // Verify Identity
    require(checkSig(recipientSig, recipientPk));  // Verify Signature
    require(sha256(secret) == secretHash);         // Verify Secret!
}
```
Because `sha256(secret) == secretHash`, the contract unlocks, and the solver receives the BCH. **No central bridge required.**

#### 2. The `refund` Function (The Safety Net)
If the solver never fulfills the intent on the destination chain, the user simply waits. Once the `timelock` (Unix timestamp) passes, the user can call `refund`:

```cashscript
function refund(pubkey senderPk, sig senderSig) {
    require(hash160(senderPk) == senderPkh);
    require(checkSig(senderSig, senderPk));
    require(tx.time >= timelock);                  // Introspection Opcode!
}
```
Using the BCH native introspection opcode `tx.time` (verifying `OP_TXLOCKTIME`), the contract mathematically guarantees the user gets their money back.

---

## 🛠️ Getting Started & Testing

This repository includes a comprehensive test suite that simulates the entire lifecycle of an HTLC. It's the perfect place for developers to understand the mechanics.

### Prerequisites
- Node.js (v18+)
- Basic understanding of [CashScript](https://cashscript.org/) concepts

### Installation

```bash
npm install
```

### Running the Test Suite

We use `cashscript` alongside `ElectrumNetworkProvider` to run tests directly against the **BCH Chipnet (Testnet)**. 

```bash
npm test
```

The test script (`test/CrossChainHTLC.test.ts`) runs through 5 distinct validations:
1. **Compilation & Instantiation**: Compiles the `.cash` file and generates a deterministic address.
2. **Secret/Hash Verification**: Ensures SHA-256 alignment between the raw 32-byte secret and the hashed lock.
3. **Timelock Logic**: Simulates before/after expiry scenarios to validate the `tx.time` constraints.
4. **HTLC Lifecycle Simulation**: A step-by-step console Walkthrough of the atomic swap process.
5. **HTLC Uniqueness**: Proves that changing the `secret` or any parameter creates a distinct, parallel covenant address.

---

## 🧠 Talking Points for Hackathons & Interviews

When talking about this implementation, highlight these technical achievements:

1. **State on a Stateless Chain**: We map complex cross-chain state machines onto BCH's UTXO model by baking the state into the covenant's constructor parameters.
2. **True Decentralization**: There is no "admin key" or "relayer multisig". The relayers in our ecosystem only observe and pass data; they cannot steal funds because they cannot fake the SHA-256 secret.
3. **Future-Proofing (2026 Upgrades)**: Connect this implementation to the upcoming May 2026 BCH VM upgrades. While we do this securely today with linear paths, the upcoming `OP_BEGIN/OP_UNTIL` and `OP_Invoke` will allow us to compress this logic and do multi-hop verifications natively on-chain. 

---

*Part of the Intents Swap Protocol Ecosystem — Bringing Intent-based DeFi to Bitcoin Cash.*
