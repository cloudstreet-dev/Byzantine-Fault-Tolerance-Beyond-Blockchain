# Permissioned Blockchains and Where They Fit

Let's be honest about the genre.

A **permissioned blockchain** is a distributed ledger where participation is restricted — you have to be authorized to run a node or submit transactions. Every participant is known, authenticated, and part of some governance arrangement. The participants may not fully trust each other (otherwise why bother), but they are not anonymous and not pseudonymous. They are named organizations with signed agreements.

This is, mechanically, the classical BFT problem. You have a known set of `N` nodes, some of which may be Byzantine, and you want to totally order a sequence of transactions. The answer is PBFT, HotStuff, or a close relative. We have met all of them.

The open question is: does calling this system a "blockchain" help, or is it marketing paint on a distributed database?

This chapter tries to answer that with specifics rather than polemic.

## The cases

The major permissioned-blockchain platforms, as of 2026:

### Hyperledger Fabric

Developed under the Linux Foundation's Hyperledger project, originally contributed by IBM. Fabric's architecture is explicitly modular:

- **Peers** execute chaincode (smart contracts) and hold ledger state.
- **Orderers** form a consensus group that orders transactions.
- The consensus algorithm in the ordering service is pluggable — historically Kafka (which is itself not BFT), then Raft (CFT), and more recently BFT variants (SmartBFT, a PBFT descendant).

Why the modularity? Because Fabric's authors recognized that "consensus on transaction order" is the classical SMR problem, and they let you pick your favorite algorithm. The rest of Fabric — smart-contract execution, privacy via channels, membership service providers — is built around the ordering service.

For most Fabric deployments, the chosen ordering algorithm is Raft (CFT). The consortium members trust each other enough that Byzantine tolerance isn't needed; they just want fault tolerance against crashes. This is telling: even in the "permissioned blockchain" space, CFT is often the pragmatic choice.

### R3 Corda

Corda is a different shape. Rather than replicating the ledger to every participant, each transaction is only visible to the parties involved. Consensus is reached via **notary services** — each notary is a Paxos, Raft, or BFT cluster that stamps uniqueness on transactions (preventing double-spending of states).

Corda's insight: in a financial consortium, you don't want every participant to see every transaction. Privacy is a first-class concern. The "blockchain" label is somewhat misleading for Corda — it's really a distributed ledger with pairwise-visible transactions and external uniqueness services. But the notary services are BFT or CFT consensus groups under the hood.

### Ethereum's permissioned variants

Quorum (originally by JPMorgan, now owned by ConsenSys) is a permissioned fork of Ethereum with pluggable consensus — originally Istanbul BFT (an IBFT, a PBFT descendant), later updated.

Besu (another Ethereum client, Apache-licensed) supports IBFT and QBFT consensus in addition to standard Ethereum PoW/PoS. Used in Hyperledger Besu and various enterprise settings.

These use Ethereum's execution layer (EVM, smart contracts, accounts) but replace the Nakamoto consensus with classical BFT. Why? Because Nakamoto consensus is expensive, slow to finality, and unnecessary when participants are known.

### Other deployments

- **Interbank networks** (e.g., the former Libra/Diem project, various bank consortia) — typically use HotStuff variants.
- **Supply chain platforms** — often Fabric-based, with chosen consensus depending on trust requirements.
- **Tokenized asset platforms** — varies widely; some are Fabric, some are Quorum, some are proprietary.

Almost invariably, the consensus algorithm in these systems is one we've met: Raft, PBFT, HotStuff, or a close descendant.

## What the blockchain framing adds

Three things that can genuinely matter, even in a permissioned setting:

1. **Smart contracts / executable state transitions.** If the application requires automated execution of business logic with agreement on outcomes, the blockchain model offers a clean abstraction. This is not inherent to consensus — you could bolt smart contracts onto any SMR system — but blockchain platforms come with the tooling.
2. **Tamper-evident log + cryptographic audit.** Blockchains chain blocks via hashes. Even if the consensus algorithm doesn't care about this (a Raft log works fine without hashes), the hash chain is a forensic feature: if you want to prove to a regulator in 2030 that a certain transaction was ordered at a certain time, signed blocks with hash links are a nice artifact.
3. **Token / asset semantics.** If the application is about transferring assets with strict conservation rules, the UTXO or account-based models of blockchain platforms are well-developed. Reimplementing them on a Raft-backed database is possible but reinventing a wheel.

If *none* of these three matter for your application, the "blockchain" label is mostly aesthetic. You could replace the system with, say, Consul or etcd plus application logic and get a simpler, faster, cheaper stack.

## What the blockchain framing adds *that is actually harmful*

To be fair in the other direction:

1. **Vocabulary mismatch with operators.** Operators of Raft/Paxos clusters have decades of shared vocabulary, runbooks, debugging tools, monitoring practices. Permissioned-blockchain teams sometimes reinvent these from scratch in blockchain terminology. A "validator timeout" is a leader election timeout; a "stuck block" is a failed view change. Rebranding doesn't help new operators.
2. **Performance budget for features that are not needed.** Hash-chaining every block, signing every transaction, maintaining Merkle trees — these have cost. If your application doesn't need cryptographic audit trails at the block level, you're paying for a feature you don't use.
3. **Regulatory cargo-culting.** "Putting it on the blockchain" sometimes appears in compliance conversations as if it solves problems it doesn't — a blockchain won't prevent a compromised operator from submitting false transactions, just like a database won't.

## The honest question

When a colleague proposes a permissioned blockchain, the productive question is: *"What does the blockchain framing give us that a distributed database with signed transactions wouldn't?"*

Good answers:

- We need to execute smart contracts in a standardized language with tooling our auditors can inspect.
- We need tamper-evident logging for regulatory reasons, with cryptographic proofs verifiable by outside parties.
- We already have blockchain infrastructure and the marginal cost of adding this workload to it is low.
- We expect the set of participants to change over time in ways that benefit from explicit on-chain governance.

Bad answers:

- Blockchains are the future.
- It's a buzzword we need for funding.
- It sounds better in the PR release.

Neither list is exhaustive, but the distinction is roughly: *does the blockchain framing earn its cost?*

## A brief note on "DLT"

"Distributed Ledger Technology" (DLT) is the umbrella term favored by people who want the benefits of the blockchain framing without some of its baggage. In practice, DLT includes:

- Public blockchains (Bitcoin, Ethereum) — out of scope here.
- Permissioned blockchains (Fabric, Corda, Quorum) — covered above.
- Distributed databases positioned as DLT (various startups).

The term is broad enough to be not very useful. If someone says "DLT," the right follow-up is "what's your consensus algorithm?" If the answer is Raft, you're running a distributed database. If the answer is HotStuff, you're running a classical BFT system. Call it what it is.

## A practical compass

Here's a rough decision procedure:

```
Is membership open to anyone?
├── Yes → You probably want a public blockchain. Out of this book's scope.
└── No ─ (permissioned)
     │
     Do you trust all operators fully?
     ├── Yes → CFT consensus (Raft, etc). Not a blockchain, just
     │        a distributed database.
     └── No ─ (some operators might be malicious)
          │
          Do you need smart contracts, tamper-evident logs, or
          token semantics as core features?
          ├── Yes → Permissioned blockchain (Fabric, Quorum, etc).
          │        The blockchain framing earns its cost.
          └── No ─── BFT consensus (PBFT, HotStuff) over whatever
                     data model you already have. The blockchain
                     framing would be overhead.
```

This is a sketch, not a law. Real decisions involve team expertise, regulatory context, vendor relationships, and operational preferences. But the core question — does the framing earn its cost? — is the useful one.

## Fair summary

Permissioned blockchains are a legitimate corner of the distributed systems space. They exist because some applications genuinely benefit from the combination of classical BFT consensus + smart-contract execution + tamper-evident audit trails, wrapped in a consistent platform.

They are also oversold, over-applied, and often confused with public blockchains by stakeholders who don't care about the difference. An engineer who understands classical consensus can cut through this confusion efficiently: ask what algorithm is under the hood, what the trust model is, and what the blockchain framing actually provides. Most of the time, you'll find yourself back in the territory of the previous eleven chapters.

With that out of the way, let's turn to what actually goes wrong in systems using these algorithms.
