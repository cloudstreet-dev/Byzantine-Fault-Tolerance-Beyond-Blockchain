# Choosing a Consensus Algorithm

You've read twelve chapters of theory and history. Someone walks up to your desk and says: "We need to replicate this service across three datacenters. What algorithm do we use?"

This chapter is the answer. Or rather, it is a procedure for producing an answer that is specific to your situation.

## Start with three questions

Before picking an algorithm, answer:

1. **Who are the participants, and do you trust them?**
2. **What is the blast radius of a failure — how strong does consistency need to be?**
3. **What is the network between participants — LAN, WAN, cross-continent?**

These three questions define the shape of the design space. Most bad consensus decisions come from picking an algorithm before answering them.

### Who are the participants?

- **Single organization, single deployment:** you trust the operators. CFT is enough.
- **Single organization, multiple teams/regions:** you mostly trust the operators but have process isolation concerns. CFT with good monitoring is usually enough.
- **Multiple organizations, known and contractually bound:** permissioned setting. CFT may still be enough if the contract and incentives align. BFT if individual nodes could be compromised or act adversarially.
- **Open participation, pseudonymous:** you are outside this book. Go read about Nakamoto consensus, proof-of-stake, and the economic-security literature.

A surprising share of real systems can honestly answer "single organization, single deployment." For those, do not use BFT. It costs more in nodes, messages, and operational complexity without buying real benefit.

### Consistency requirements

Strong consistency (linearizability) via consensus is expensive. Decide if you need it:

- **Linearizable:** operations appear instantaneous, in a real-time-respecting order. The model most "consensus databases" promise. Costs: one round-trip per write, latency pauses during partitions.
- **Sequential / serializable:** total order, but not necessarily respecting real-time across clients. Cheaper than linearizable.
- **Causal:** preserves happens-before. Good enough for many collaborative applications (document editing, chat).
- **Eventual:** reads might be stale; writes eventually propagate; conflicts resolved by CRDTs or application-level rules. Cheapest; highest throughput; hardest to reason about.

If eventual consistency is enough, you may not need consensus at all. Dynamo-style systems (Cassandra, Riak, DynamoDB in default modes) are the alternative. Consensus-based systems trade latency and availability for stronger guarantees. Know which you need.

### Network realities

- **Within a datacenter / LAN:** single-digit ms RTTs, low variance. Raft or Paxos with short timeouts work well.
- **Cross-AZ within a region:** still single-digit ms, some variance. Fine for most consensus systems.
- **Cross-region, same continent:** 30–100ms RTT. Consensus still viable but every write pays this once. Consider whether writes need to synchronously hit multiple regions.
- **Cross-continent:** 100+ ms RTT. Cross-continental consensus for every write is painful. Consider sharding by region, using follower reads, or accepting asynchronous replication.

A five-node cluster where two nodes are in Sydney and three in New York is a cluster where every decision needs a Pacific crossing. Pay attention to where quorums live.

## The decision procedure

A rough flowchart:

```
Do you trust all participants?
  Yes: Use CFT.
  No:  Do you have smart contracts, audit, token semantics?
         Yes: Permissioned blockchain (PBFT/HotStuff-backed).
         No:  BFT consensus directly (PBFT for small n, HotStuff for large n).

In CFT, pick based on team experience and ecosystem:
  Team already knows Raft: use Raft. (etcd's raft library, hashicorp/raft.)
  Team knows Paxos or has Paxos-shaped code: use Multi-Paxos.
  Neither: use Raft.

Does your data need to scale horizontally?
  Yes: Multi-raft (partition data, one Raft group per partition).
       CockroachDB, TiKV, some in-house patterns.
  No:  Single-group Raft/Paxos. (etcd, Consul, ZooKeeper-pattern.)

Cross-region?
  If yes, and writes need to be linearizable globally:
    - Spanner-style (TrueTime + Paxos) if you have the clock infra.
    - Otherwise: accept the WAN-RTT write latency, or shard by region.
  If yes, but writes are region-local with occasional cross-region reads:
    - One consensus group per region, application-level handling.
```

## When crash-fault tolerance is enough

For most working engineers, the honest answer is "CFT is enough." This includes:

- Operational metadata (cluster state, service discovery, feature flags): etcd, Consul, ZooKeeper.
- Traditional databases needing HA: any Raft/Paxos-backed system (CockroachDB, FoundationDB, TiDB, etc).
- Message queues and log systems: Kafka (KRaft), Pulsar, Apache Pulsar's BookKeeper.
- Distributed locks and leader elections for larger applications: built on top of ZooKeeper/etcd/Consul.

The node count is `2f+1`, the algorithm is Raft or Paxos, and the algorithm choice is less important than picking a battle-tested library and operating it well.

## When Byzantine fault tolerance is actually needed

BFT is justified when at least one of these is true:

- **Multi-organization trust boundary.** A consortium where individual nodes could be compromised or misbehave independently.
- **Regulatory or audit requirements.** Where cryptographic non-repudiation of every decision is a feature, not just a side effect.
- **High-integrity systems.** Financial settlement, government record-keeping, safety-critical coordination, where undetected corruption is unacceptable at any rate.
- **Known adversarial environment.** Multi-tenant systems where you *have* reason to believe some participants will act against the group's interest.

BFT is *not* justified merely because:

- "We might get hacked." (Defense in depth > BFT; BFT doesn't help if the attacker controls `f+1` nodes.)
- "We want to be really safe." (CFT is very safe. BFT is safer against specific failure classes that may not apply.)
- "We might deploy on untrusted hardware." (Usually better addressed with attestation and tooling than with BFT.)
- "Blockchain-like, but permissioned." (Maybe. See Chapter 12.)

## When eventual consistency beats consensus

Consensus is not the right answer for:

- **Shopping cart state.** Last-write-wins or merge semantics are fine.
- **Social media feeds.** Ordering can be approximate; occasional staleness is tolerable.
- **Analytics pipelines.** Throughput matters more than strict consistency.
- **Cache tiers.** You can always recompute; staleness is just a cache miss with higher latency.
- **Systems where writes conflict rarely.** CRDTs and eventual consistency are cheaper and simpler.

A common mistake is to shove all these into a consensus-backed database because the team already runs one. If the semantics don't require consensus, you are paying for consistency you don't use — and the system might be less available for unrelated reasons.

## How to evaluate a consensus claim

When you receive a system and a vendor or author claims it "uses Raft" or "achieves linearizability" or "has Byzantine fault tolerance" — how do you evaluate?

A short checklist:

1. **What is the exact algorithm?** Raft / Paxos / VR / PBFT / HotStuff / custom? If custom, is there a paper or spec?
2. **What is the failure model?** Crash, crash-recovery, or Byzantine? Does the deployment reality match?
3. **What is the quorum size?** Does it match the claimed fault tolerance (`2f+1` for CFT, `3f+1` for BFT)?
4. **Does it `fsync` before acknowledging?** If no, the durability claim is probably a lie.
5. **How does it handle network partitions?** Does a minority partition continue to serve reads? Writes? Does it refuse?
6. **Has it been Jepsen-tested?** If yes, what issues were found and have they been fixed? If no, ask why.
7. **What are the read semantics?** Linearizable reads? Bounded-staleness? Best-effort?
8. **How does it handle clock skew?** Are any safety guarantees dependent on clocks?
9. **What is the leader election / view change strategy?** How long can a cluster be leaderless?
10. **What happens in a rolling upgrade?** Can the cluster tolerate one node at a time being down, for the time it takes to upgrade?

If the vendor cannot answer most of these, you are not getting a consensus system — you are getting a product with consensus-shaped marketing. Caveat emptor.

## When in doubt

A few defaults that rarely lead you wrong:

- **For new CFT work:** Raft with a battle-tested library (etcd's raft, hashicorp/raft, or equivalent in your language). Five-node cluster, colocated in one region, multi-AZ. Linearizable reads via leader with read index. `fsync` on every commit.
- **For BFT work:** HotStuff or a direct descendant. Don't write your own.
- **For eventually consistent work:** CRDT-based approach if the semantics admit it; otherwise, a battle-tested AP system (Cassandra, DynamoDB with eventual consistency).
- **For cross-region:** think twice; is regional sharding viable?

The reason these defaults are safe is that they represent enormous accumulated engineering effort. Choosing them instead of rolling your own isn't a cop-out; it is the right choice for almost everything.

## A final word on consensus and humility

In writing this book, I've come to believe that the main thing distributed systems teach is a kind of humility. Each of these algorithms was designed by smart people over careful years. Each has been implemented dozens of times; most implementations had subtle bugs; many had catastrophic ones. The literature is a long chain of "we thought this was right, then we found a case where it wasn't."

If you are building a consensus system from scratch, you are joining this chain. Most likely you will fail in ways you cannot foresee. The antidote is not caution to the point of paralysis but genuine respect for the problem: read carefully, test exhaustively (TLA+, Jepsen, fault injection), and have experienced reviewers catch what you miss.

The algorithms here have earned their place by surviving decades of adversarial scrutiny. Start with one of them.
