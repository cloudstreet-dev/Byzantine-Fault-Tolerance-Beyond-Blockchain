# Liveness, Safety, and the Things That Go Wrong

In distributed systems, correctness has two faces.

- **Safety** — nothing bad ever happens. No two nodes commit conflicting values; no committed write is lost; no client sees state that violates the contract.
- **Liveness** — something good eventually happens. Requests eventually get responses; leaders eventually get elected; snapshots eventually get taken.

These are orthogonal. Safety alone is easy (never decide anything, and you never decide wrong). Liveness alone is easy (decide something fast, correctness be damned). Systems are hard because they want both at once.

This chapter is about where real systems fail on each axis — with enough concreteness that you recognize the shapes when you see them in your own logs.

## Safety vs. liveness, operationalized

Leslie Lamport, again, gave us the formal distinction:

- A **safety property** is one whose violation can be observed in a finite execution. If a system ever has two leaders at the same term, that's a safety violation — a specific bad moment in time.
- A **liveness property** is one whose violation requires an infinite execution to observe. "The cluster eventually elects a leader" — you can't rule it out by looking at any finite run; maybe it just needs more time.

In practice, safety violations are *worse*. A safety violation is a bug that corrupted state or let inconsistent data through. It cannot be fixed by waiting; someone has to reconcile. A liveness failure is a stall — bad, but recoverable by the system resuming forward motion.

Production systems prioritize safety. An etcd cluster that stalls for 30 seconds during a bad election is annoying; an etcd cluster that lets two clients both win a lock is catastrophic. Every consensus algorithm in this book is built to sacrifice liveness before safety.

## The common failure modes

### Split brain

Two leaders believe they are in charge simultaneously and both serve writes. Causes:

- Stale leader that hasn't yet discovered a new term / view.
- Network partition where both sides elected a leader.
- Clock skew fooling a leader lease into looking valid when it's not.

All modern consensus algorithms have specific defenses:

- **Quorum intersection.** A partition can only elect a leader on a side that has a majority. The other side can't. So at most one "real" leader at any term, even if the old one is still alive and unaware.
- **Term / view numbers.** The stale leader's RPCs are rejected because they carry the old term. Once it hears about the new term, it steps down.
- **Lease fences.** If a leader relies on a time-bounded lease, the lease expires before a new leader can be elected (by design), so the old one stops serving.

Split brain in a *correctly implemented* Raft is impossible. Split brain in a Raft that skips `fsync` or mishandles votes is absolutely possible. This is why the boring implementation details from Chapter 5 matter.

### The stale read

Related but distinct. A leader thinks it's still the leader, serves a read from its local state, but has actually been superseded. The returned data is stale.

Defenses:

- **Read index.** Leader sends a heartbeat to confirm it still holds majority, then serves the read.
- **Leader lease + synchronized clocks.** Leader holds a lease; during the lease it's safe to read locally. Needs clock safety.
- **Read from quorum.** Read from majority; take the most recent. (Paxos-style read.)

Choosing here is a latency-vs-consistency dial. Production systems differ.

### Leader flapping

Leaders keep getting elected and then losing their position in quick succession. Causes:

- Election timeout tuned too aggressively for the network's variability.
- Partial partitions where different subsets of nodes see different "reachable" leaders.
- GC pauses on leaders (JVM systems especially) making them look dead for a moment, triggering an election, then coming back.

Symptoms: log churn, write latency spikes, reduced throughput. The cluster is *technically* correct — each elected leader is valid — but each election costs real time and the system never stabilizes.

Fixes:

- Longer election timeouts (at the cost of slower recovery from real failures).
- Leader lease-based heartbeats (makes brief unreachability less likely to cause elections).
- Better GC tuning.
- Network fixes (partial partitions need dedicated effort).

### Cascading failure

One node fails; the remaining nodes pile on load; a second node fails under the load; the quorum breaks; the cluster stalls. The classic "thundering herd" variant.

Cascading failures in consensus systems tend to involve:

- Retrying clients hammering the new leader.
- Snapshot transfers overloading the network of a newly-caught-up replica.
- Unbounded memory growth in leader election storms.

Defenses:

- **Backpressure.** The leader should refuse requests when overloaded, not queue them.
- **Rate limiting on catch-up.** Snapshot streaming should respect bandwidth budgets.
- **Jitter.** Clients retrying a failed write should add randomized delay.
- **Capacity planning.** The cluster should be sized so that `N-1` nodes can handle peak load with headroom — not `N` at redline.

### Clock skew

Spanner makes clock skew a first-class concern. Most other systems assume clocks are "close enough" and are surprised when they aren't.

- **NTP skew** in most cloud environments is sub-10ms but occasionally jumps.
- **Clock jumps** happen: VM migrations, NTP corrections, leap-second handling gone wrong.
- **Clocks running fast** on one node vs. another — rare but real.

Failure modes:

- A leader's lease looks longer than it actually is; a new leader is elected before the old one has stepped down.
- Timeouts fire early or late.
- Audit logs claim impossible orderings.

If your algorithm claims safety without reliance on clocks (Raft/Paxos/etc proper), clock skew should only cause liveness issues — leader flapping, slow elections. If safety depends on clocks, you are one VM migration away from a split brain.

### Disk failure and "fast" fsync

The quiet safety killer.

- `fsync` is expensive; under load, implementers are tempted to batch or skip it.
- Cloud block storage (EBS, GCE persistent disks) has its own journey from "application fsync" to "bytes durable," and various failure modes in between.
- A storage device that lies about its sync behavior (some consumer SSDs do this, under the rubric of "write caching") violates the protocol's assumptions.

If disk A returns from `fsync` before the data is durable, and then the node crashes, the on-disk log is not what the protocol thinks it is. On recovery, the node may contradict its own past actions — a Byzantine failure from the protocol's perspective, even if no nodes are malicious.

Jepsen has caught several products doing this. The general guidance:

- Use `fsync` (or `fdatasync`) religiously on the critical path.
- Understand your storage layer's durability semantics.
- Run Jepsen-like tests against your storage choices.

### The "I'm healthy" lie

Node A is up but can't reach node B. Node B is up but can't reach A. Both report themselves healthy to external monitoring. The cluster is reporting no failures but not making progress.

This is the partial partition problem. Health checks are usually "am I up and responsive to the check"; they don't tell you "am I able to participate in consensus." Sophisticated monitoring tests consensus progress directly — "is the leader making forward progress?" — which is better, if you have it.

## Real post-mortems that illuminate the theory

### MongoDB rollback incidents

MongoDB's replication layer has been the subject of several Jepsen reports ([jepsen.io/analyses/mongodb-4.2.6](https://jepsen.io/analyses/mongodb-4.2.6) among them) showing scenarios where committed writes could be lost under specific failure patterns and configurations. The fixes in subsequent versions bring it closer to conventional Raft-like behavior, but the history is a good reminder that "we have a replication protocol" is not the same as "it's correct under adversarial failures."

### etcd disk latency incidents (Kubernetes outages)

Various Kubernetes outage write-ups attribute cluster-wide failures to etcd struggling under disk latency — slow commits cause slow API server responses, which cause slow controller reconciliations, which cascade into application-layer problems. The root-cause in these is rarely a bug in etcd's Raft — it's the operational assumptions around disk performance not being met. (Look at public Kubernetes post-mortems and vendor blog posts for specific cases.)

### The great GitHub MySQL outage of 2018

Not a consensus system exactly, but instructive. A brief network partition triggered an automated failover; the cross-region replication state was not what the failover system assumed; inconsistencies ensued. GitHub's [public post-mortem](https://github.blog/2018-10-30-oct21-post-incident-analysis/) documented 24 hours of degraded service for what started as a 43-second network blip.

Lessons from this kind of incident:

- Automated failover is safety-critical and can make things worse if the model is wrong.
- Cross-region replication is partial synchrony at best, and tooling should know that.
- Post-incident cleanup (reconciling the diverged state) took vastly longer than the triggering event.

### Kyle Kingsbury's Jepsen series

Kingsbury's Jepsen testing ([jepsen.io](https://jepsen.io)) has found consistency bugs in essentially every major distributed system: Riak, MongoDB, Elasticsearch, Aerospike, VoltDB, CockroachDB, YugabyteDB, etcd (minor), Consul, ZooKeeper, FoundationDB (passed, notably), Kafka, Redis, DynamoDB, and many more.

The common themes across Jepsen reports:

- Claims of linearizability that are violated under specific timing.
- Default settings that silently lose data under network partitions.
- Retry and deduplication logic with edge cases.
- Storage layer assumptions that break in cloud environments.

Every practitioner should read 5–10 Jepsen reports cover to cover. They teach the gap between "we implemented Raft" and "our system is safe" better than any textbook chapter.

## What to take from this chapter

1. **Safety and liveness are different properties.** Safety violations corrupt data; liveness failures stall progress. Systems are designed to sacrifice liveness before safety.
2. **Most production issues are operational, not algorithmic.** The algorithm is usually correct on paper. The deployment, the tuning, and the environment are where the bugs live.
3. **The same shapes recur.** Split brain, stale read, leader flap, cascading failure, clock skew, disk lies. Recognize them early; instrument for them; test for them.
4. **Jepsen is the practitioner's school.** Read reports. Apply the mindset to your own systems.

With safety, liveness, and failure modes in hand, we can finally turn to the practical question: given all this, how do you *pick* a consensus algorithm?
