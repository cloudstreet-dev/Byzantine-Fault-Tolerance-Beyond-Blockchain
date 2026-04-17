# Real Production Systems

The algorithms in the preceding chapters are the theory. Now: what happens when you run one of them for a living?

This chapter walks through the production systems you are most likely to encounter, what algorithm each uses, the choices that shaped the implementation, and what practitioners have learned about operating them. All claims are dated to 2026; consensus systems evolve, and the landscape will have shifted by the time you read this.

## etcd

Algorithm: **Raft**.

[etcd](https://etcd.io) is a distributed key-value store written in Go, originally by the CoreOS team and now a CNCF project. It is best known as the backing store for Kubernetes — the cluster's entire desired-state configuration lives in etcd. When etcd is unhappy, your Kubernetes cluster is unhappy.

Implementation choices:

- **raft library split out.** The `go.etcd.io/raft` library is independent of etcd itself — it implements only the Raft state machine, leaving I/O, storage, and networking to the caller. This design is what let CockroachDB, TiKV, and dozens of others adopt Raft quickly: they reused etcd's raft library rather than implementing from scratch.
- **Multi-raft** (in CockroachDB, TiKV) isn't in etcd itself. etcd is single-group Raft; if you want to horizontally scale, you go elsewhere.
- **MVCC with persisted log.** The log (append-only) and the MVCC key-value state live together. Reads are served from the leader by default; there's a `linearizable` flag for reads that forces a round-trip-to-quorum; there's a `serializable` flag for faster but stale reads.
- **Compaction.** History is retained for a configurable window; old revisions are compacted on a schedule. A running cluster has to keep up with compaction or the log fills disk.

Operational lessons:

- **Three-node clusters are fragile.** They tolerate one failure, but during maintenance (rolling updates), you have no failure tolerance at all. Five-node clusters are the production norm for anything important. Kubernetes documentation itself recommends five for larger clusters.
- **Disk latency is everything.** Raft does an `fsync` per committed entry. If your disk has 10ms commit latency, your writes have 10ms commit latency. SSDs, preferably NVMe. Avoid networked block storage with bad tail latency.
- **Don't run on spinning disks.** Really.
- **Watch carefully at scale.** Kubernetes' etcd guidance recommends keeping etcd's data size under a few gigabytes; beyond that, leader elections and snapshots get painful. Periodic defragmentation is part of ops.

Known incidents: the [Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/) and various etcd operator guides document etcd failure modes that real clusters hit. Post-mortems on Kubernetes clusters losing etcd quorum are a recurring genre.

## Consul

Algorithm: **Raft**.

[Consul](https://www.consul.io) by HashiCorp provides service discovery, health checking, KV storage, and service mesh functions. Its central data store is Raft-replicated.

Implementation choices:

- **Its own Raft implementation** in the `hashicorp/raft` Go library, independent of etcd's. Feature-compatible, different code. HashiCorp's library is also widely used beyond Consul.
- **Servers and clients.** Consul has two roles: servers (participate in Raft) and clients (agents that connect to servers, do health checks, register services). Most Consul deployments have 3 or 5 servers and many clients.
- **Multi-datacenter.** Consul supports federation across datacenters, with each datacenter running its own Raft cluster and cross-datacenter gossip for service discovery.
- **Network coordinates.** An interesting non-Raft bit: Consul maintains a Vivaldi network coordinate for each node, used to estimate RTTs for nearest-neighbor routing. Not consensus, but worth mentioning as an example of the kinds of features that sit *on top of* consensus.

Operational lessons:

- **Leader election is sensitive to network hiccups.** Cross-region Consul federations have seen leader flapping under WAN latency variability. The answer is usually "don't stretch a single Raft cluster across regions."
- **Gossip isn't consensus.** Consul's cross-datacenter membership uses a gossip protocol (Serf), which is eventually consistent — don't treat it as if it were.

## ZooKeeper

Algorithm: **Zab** (ZooKeeper Atomic Broadcast).

[Apache ZooKeeper](https://zookeeper.apache.org) is the elder statesman — released by Yahoo in 2007, with roots in the Chubby lock service work at Google. It predates Raft and was the default "consensus thing" for a decade.

Zab is a Paxos cousin — not quite Paxos, not Raft, but in the same family. Its normal-case operation is a two-phase protocol with a leader; its leader-election protocol is distinct but conceptually similar to view change. Zab's explicit goal is total order on broadcast messages, not general-purpose consensus — though the difference is small.

Implementation choices:

- **JVM.** ZooKeeper is Java. Bringing a JVM dependency into your infrastructure is a nontrivial choice.
- **ZNodes.** The data model is a hierarchical namespace of "znodes," each with a small payload. Watches on znodes are a key feature: clients register interest, get notified of changes.
- **Sessions and ephemeral nodes.** A client holds a session; ephemeral znodes exist only while the session is active. This is how ZooKeeper underlies distributed locking and leader election for *other* systems.
- **Observers.** Non-voting replicas that serve reads without participating in consensus. Useful for scaling reads without inflating the quorum.

Operational lessons:

- **ZooKeeper was complicated to run.** You needed to tune JVM heap, GC, session timeouts, transaction log sync, snapshot timing — and different applications wanted different configurations.
- **Clients had bugs that looked like ZooKeeper bugs.** A common production pattern: app hits some weird behavior, blames ZooKeeper, actually the client library's session handling.
- **It worked.** For all the complexity, ZooKeeper clusters ran for years in production. It's not fashionable anymore, but it's not broken.

## Kafka: ZooKeeper to KRaft

For its first decade, Apache Kafka used ZooKeeper to store cluster metadata — topic configurations, partition assignments, broker membership. This coupling was a major operational burden: you had to run (and understand) ZooKeeper just to run Kafka.

In 2020, KIP-500 proposed replacing ZooKeeper with a built-in Raft quorum — **KRaft**. The idea: Kafka already knew how to replicate logs (that's what Kafka *is*); the ZooKeeper dependency was a historical accident. Using an internal Raft quorum to store metadata eliminates the external dependency.

- **KRaft** became production-ready around Kafka 3.3 (2022). ZooKeeper-based Kafka was formally deprecated; Kafka 4.0 (2025) removed ZooKeeper support entirely.
- The metadata quorum in KRaft is usually 3 or 5 *controllers*, separate from the data brokers.
- The transition was multi-year and had extensive compatibility concerns.

The KRaft migration story is an interesting example of a system *de-coupling* from consensus: Kafka went from "ZooKeeper is a hard dependency" to "we embed our own Raft." Many large users preferred the simpler operational model — one system to understand, not two.

## Google Spanner (Paxos + TrueTime)

[Spanner](https://research.google/pubs/pub39966/) is Google's globally-distributed, externally-consistent database. Its consensus mechanism is Paxos — but the interesting bit is the *clock*.

Spanner's shards are each a Paxos group (typically 5 replicas across regions). Transactions are ordered using **TrueTime** — an API that returns a time interval `[earliest, latest]` guaranteed to contain the true current time. Google achieves tight TrueTime bounds (a few milliseconds) with GPS and atomic clocks in every datacenter.

Why this matters: with reliable time, Spanner can *commit-wait* to ensure that timestamp ordering respects real-time ordering. External consistency (also called "strict serializability") comes out the far side.

Implementation specifics:

- Paxos is used within each shard for replication.
- **Two-phase commit** across shards (for multi-shard transactions).
- **TrueTime** is the novel piece — GPS + atomic clocks + a custom API returning intervals, not points.

Spanner's architecture is unusual, and Google made the hardware investment to make it work. Knock-offs (CockroachDB, YugabyteDB, TiDB) use variations of Paxos/Raft without TrueTime-level clocks, trading external consistency for operational simplicity.

## CockroachDB

Algorithm: **Multi-Raft**.

[CockroachDB](https://www.cockroachlabs.com) is a SQL database inspired by Spanner but without TrueTime. Instead of TrueTime, it uses HLC (Hybrid Logical Clocks) and a "commit wait" equivalent that is looser than Spanner's.

- **Ranges.** Data is split into 512 MB (by default) ranges; each range is a Raft group.
- **Multi-raft.** A single node participates in many Raft groups — thousands, at scale — one per range-replica. The `go.etcd.io/raft` library is the foundation.
- **Leaseholder.** Each range has a leaseholder, a replica that serves reads and coordinates writes without a Raft round-trip for every read. Leases are time-bounded.

Engineering lessons:

- **Multi-Raft at scale is engineering-heavy.** Thousands of Raft groups means thousands of heartbeats, leader elections, snapshots. CockroachDB has invested heavily in heartbeat coalescing and snapshot streaming.
- **Leaseholder placement matters.** If your reads are cross-region but the leaseholder is somewhere else, you pay WAN latency. Follower reads (with bounded staleness) are an optimization here.
- **Range splits and merges.** As data grows or shrinks, ranges are dynamically split/merged. This is a consensus-level operation and needs care.

## FoundationDB

Algorithm: **Custom** — related to Paxos, but with a distinct architecture.

[FoundationDB](https://www.foundationdb.org), originally a startup, now an Apple project and open source, is an ordered key-value store with strict serializability. Its architecture separates:

- **Coordinators** — a small, stable set that run consensus over cluster configuration.
- **Proxies** — accept client writes.
- **Resolvers** — detect conflicts.
- **Log servers** — durable log.
- **Storage servers** — serve reads.

The design emphasizes *testability*. FoundationDB has a deterministic simulator that can replay the entire cluster under adversarial timing and failure injection. This enabled development confidence that is unusual in the field.

## Aerospike, Cassandra, DynamoDB: the eventually-consistent alternative

Not every distributed database wants strict consistency.

- **Cassandra** (Dynamo-style, AP) — tunable consistency, eventually consistent by default. No consensus per write in the strict sense; reads and writes use quorum counts to approximate consistency. `QUORUM` reads+writes give you consistency at the cost of write latency, but still not linearizable.
- **DynamoDB** (AWS) — publicly describes itself as highly available and partition-tolerant, with a choice of strong or eventual consistency on reads. Internal architecture uses Paxos for leader election of partitions but is not a traditional SMR system.
- **Aerospike** — focuses on very-low-latency access, with its own consistency mechanisms that can be configured for "strong consistency" (using internal consensus for partition assignment) or for eventual consistency (trading consistency for latency).

The tradeoff these systems accept: consistency is something you dial up or down per operation, not something the database guarantees globally. For high-throughput workloads where occasional inconsistencies are manageable by the application, this is a very reasonable choice.

This is not the same tradeoff as "use a Raft-backed database and accept the latency." It is a different *model*, one where linearizability is not on the table even in the happy path. For a social media feed, fine. For your bank account, probably not.

## Observations on production reality

After walking through these systems, a few patterns emerge:

1. **Nearly all mainstream systems are CFT, not BFT.** etcd, Consul, ZooKeeper, Kafka/KRaft, Spanner, CockroachDB, FoundationDB — all crash-fault tolerant, not Byzantine. BFT is reserved for permissioned blockchains (Chapter 12) and some government/defense systems.
2. **Raft dominates new designs.** Among systems designed post-2014, Raft is the default. Paxos variants persist in older systems (ZooKeeper/Zab, Spanner) and in places where the team had a specific reason to go custom.
3. **Multi-raft is the scaling pattern.** Single-group consensus doesn't scale horizontally; partitioning your data and running one consensus group per partition does. The complexity is in managing the many groups coherently.
4. **Storage and networking matter more than the algorithm.** A Raft cluster with bad disks is a slow Raft cluster. A Paxos cluster with saturated NICs is a slow Paxos cluster. The algorithm is often the cheapest part of the system's latency budget.
5. **Observation tools are lagging.** Operators of these systems are often flying blind on internal consensus state. Better introspection — "why did leader election take 4 seconds?" — is ongoing work.

We turn next to the systems that call themselves blockchains but look a lot like the systems in this chapter.
