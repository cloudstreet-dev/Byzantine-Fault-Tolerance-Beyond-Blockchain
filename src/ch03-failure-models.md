# The Failure Models You Actually Face

Choose a consensus algorithm without being clear about your failure model, and you will pay for it. Either you will buy an algorithm that is more paranoid (and more expensive) than you need, or — worse — you will trust one that makes assumptions your environment violates, and it will betray you at the worst possible moment.

This chapter is an attempt to make you specific about what "a node fails" actually means, because the algorithms differ wildly in what they promise under different flavors of failure.

## The spectrum, from least to most adversarial

The classical taxonomy, roughly in order of increasing nastiness:

1. **Crash-stop (fail-stop)** — a node either works correctly or stops entirely. When it stops, it never sends another message.
2. **Crash-recovery** — a node can stop, then come back later, possibly with its persistent state intact.
3. **Omission** — a node is running, but some of its messages are lost (send-omission) or ignored (receive-omission).
4. **Byzantine** — a node may do anything: lie, collude, replay old messages, fabricate signatures it hasn't the keys for (if the model allows), or behave correctly just long enough to be trusted before betraying.

There is also the theoretical purity of **fail-silent** (you can't tell if a node stopped or is just slow) vs. **fail-signal** (there's some oracle that tells you a node is dead). Almost all real systems are fail-silent — dead and slow are indistinguishable over the network, which is why timeouts are hard.

Let's walk each one with the detail a practitioner needs.

## Crash-stop

**What it assumes:** nodes are correct until they aren't. "Not being correct" means halting — ceasing to send or receive messages, forever, from that point on.

**Why it's the bedrock of the field:** most of the crash-fault-tolerant algorithms (Paxos, Raft, VR) were originally specified in this model. It is simple, it is analytically clean, and it yields clean quorum arithmetic: to tolerate `f` crash failures, you need `2f+1` nodes, because a majority quorum (`f+1`) always intersects any other majority in at least one live node.

**Why it is almost always a lie in practice:** real nodes don't crash-stop. Real nodes have OOM killers. Real nodes get STW garbage collection pauses that make them look dead for thirty seconds and then wake up. Real nodes get their clocks jumped forward an hour by NTP. Real nodes get their TCP sessions killed by a cloud provider's idle timeout but keep processing. The model assumes an honest, simple death. Production failures are messy.

So why do we use it? Because for many failure scenarios, crash-stop is a *sound approximation* if you build the system carefully. Specifically, if:

- You **fence** stale leaders (so a leader who thought they were the leader, took a pause, and woke up has no authority).
- You **persist** state before sending messages based on it, so a recovered node doesn't contradict its own past.
- You **restart** pathological nodes rather than trusting them to self-heal.

...then you can treat misbehaving-but-not-malicious nodes as crashed without losing correctness. This is why production crash-fault systems are more paranoid in practice than the model suggests.

## Crash-recovery

**What it assumes:** nodes may crash and later resume. If they resume, they come back with persistent storage intact. No persistent storage loss is allowed in the basic form; variants consider *amnesia* (coming back with nothing) as a separate failure class.

**Why it matters:** real servers reboot. They may be down for seconds, minutes, or hours. If your protocol is only correct under "once a node is gone, it stays gone," you are in trouble.

**What this adds to the algorithm:**

- The recovering node has to reconstruct enough state to participate safely. It cannot assume its earlier peers still hold its in-flight messages.
- The protocol must be **idempotent** with respect to retried messages — the same message arriving twice should not cause conflicting state.
- Persistent storage must survive the crash. This usually means `fsync`, which is the most misunderstood system call in the distributed-systems world. If you don't `fsync` before acknowledging a write, you've lied about durability.

Most modern implementations (etcd, ZooKeeper, Consul) operate in the crash-recovery model, not the idealized crash-stop model. They persist to disk aggressively and the protocol is structured so that a recovering node catches up rather than corrupting the group.

### A word on fsync

`write(2)` to a file in Linux does not commit data to disk. It hands the bytes to the kernel's page cache. The kernel will eventually write them to the storage device, but "eventually" is measured in seconds, and a crash in the meantime loses the data. To make a write durable, you must call `fsync(2)` (or `fdatasync`), which forces the kernel to flush the page cache and wait for the device to acknowledge.

`fsync` is slow — milliseconds per call, sometimes tens of milliseconds on networked storage. Every consensus implementation makes some choice about how often to `fsync`. The safe choice is "before acknowledging any state change." The fast choice is "batch several state changes and fsync once." Most production systems batch; the batching window is a tuning parameter with real durability tradeoffs.

If a system claims to be "Raft" but doesn't `fsync` before acknowledging, it is running a different protocol than Raft and will, under the right conditions, lose committed writes. This is not a hypothetical — it is one of Kyle Kingsbury's greatest hits in the Jepsen testing series.

## Omission

**What it assumes:** nodes are running, but messages they send or receive can be dropped.

**Why it sits between crash and Byzantine:** a node experiencing send-omission looks *externally* like a node that is sometimes dead and sometimes alive — its peers see inconsistent behavior. But internally, the node is not malicious. It just can't reliably communicate.

Send-omission is the natural model for a node behind a flaky link. Receive-omission is the natural model for a node whose inbox is dropping packets. In practice, TCP hides much of this from you, at the cost of translating omissions into long delays, which you get to resolve with timeouts.

**Why omission matters:** a surprising number of production incidents look like omission, not like crash. A node is still "up" (it shows as healthy, its process is running) but a subset of its peers can't reach it. Partial partitions are the worst kind, because they confuse failure detectors that were designed for clean crash-stop. If node A can reach B but B can't reach C, whose perception is correct?

Crash-fault-tolerant algorithms generally tolerate omission failures gracefully — a node that can't communicate effectively looks like a dead node to the quorum, and the protocol proceeds without it. The algorithms don't have to special-case omission; they inherit tolerance from their crash model. But the operational reality is rough: partial partitions cause oscillation, leader flapping, and cascading issues that we'll see in Chapter 13.

## Byzantine

**What it assumes:** a faulty node may do anything. Send messages out of order. Lie about its state. Send different things to different peers. Collude with other faulty nodes. Replay old messages. Claim to have signatures it doesn't. Act correctly for a while, build reputation, then betray it.

**Where the name comes from:** Lamport, Shostak, and Pease's 1982 paper *"The Byzantine Generals Problem,"* which cast the problem as generals besieging a city, some of whom might be traitors. A memorable frame with staying power; we'll dig into the paper in Chapter 7.

**Why you might care:**

- **Software bugs.** A corrupted node sending wrong data is, from the protocol's perspective, Byzantine. Cosmic rays, bad memory, disk corruption, kernel bugs — real, occasional, and arbitrarily bad.
- **Compromised nodes.** If an attacker has root on one of your replicas, they can make it lie. For high-integrity systems (financial, governmental, safety-critical), this is a real threat.
- **Low-trust environments.** Multi-organization systems where you don't fully trust your co-operators. Cross-bank settlement, government/contractor arrangements, consortium networks.
- **Public networks.** Blockchains, obviously. But also any system where participants have incentives to cheat.

**What Byzantine tolerance costs you:**

- **Node count.** `3f+1` instead of `2f+1`. To tolerate 1 Byzantine failure you need 4 nodes, not 3. (Chapter 7 shows why.)
- **Message complexity.** Classical BFT protocols (PBFT) use `O(n^2)` messages per decision, vs. `O(n)` for Raft. At `n=4`, the difference is 16 vs. 4; at `n=100`, 10,000 vs. 100. Modern protocols (HotStuff) bring this down, but not for free.
- **Cryptography.** You need signed messages, not just authenticated ones. MACs aren't enough when nodes may lie about what they received — you need non-repudiable signatures, which are computationally expensive.
- **Engineering complexity.** BFT systems are harder to implement correctly than CFT systems, and harder to debug when they misbehave.

If you can avoid Byzantine tolerance, you should. If you cannot, the later chapters are for you.

## What real systems actually assume

A rough survey, as of 2026:

| System | Failure model | Notes |
|---|---|---|
| etcd | Crash-recovery | Raft, fsync before ack. |
| Consul | Crash-recovery | Raft, similar to etcd. |
| ZooKeeper | Crash-recovery | Zab, a Paxos cousin. |
| Kafka (KRaft) | Crash-recovery | Raft variant; replaced ZooKeeper dependency. |
| Spanner | Crash-recovery | Paxos per shard; external consistency via TrueTime. |
| CockroachDB | Crash-recovery | Raft per range. |
| FoundationDB | Crash-recovery | Custom protocol; SMR-style. |
| Hyperledger Fabric | Depends on ordering service; Raft (CFT) by default, BFT variants exist | Permissioned. |
| Diem/Aptos | Byzantine | HotStuff family. |
| Public L1 blockchains | Byzantine + Sybil | Not our subject. |

The vast majority of production systems you will encounter assume **crash-recovery** and nothing worse. Byzantine-tolerant consensus is rare in mainstream infrastructure — it is reserved for cases where the operational assumption "we trust the operators of each node" no longer holds.

## Choosing your failure model

The right question is not "what failures could possibly happen?" but "what failures am I willing to let my algorithm assume away?"

- If you operate all the nodes, in datacenters you control, running binaries you built: **crash-recovery** is usually sufficient. Corruption happens, but rarely, and checksums at the storage layer catch most of it.
- If you operate the nodes but cross multiple administrative domains (multi-cloud, multi-region with different teams), you may want detection for non-crash failures, but not necessarily BFT consensus. A CFT system plus good monitoring plus checksums usually suffices.
- If nodes are operated by *different* organizations that don't fully trust each other: **BFT** starts to earn its cost.
- If participation is open and you can't enumerate participants: you are outside this book's scope. Go build a blockchain.

## What to take into the algorithm chapters

When you read Chapter 4 (Paxos) and beyond, keep asking:

- What failure model does this algorithm assume?
- What quorum size does it need, and why?
- What happens if the model is violated — does the algorithm fail safe, or does it go arbitrarily wrong?

These questions are not hostile. Every algorithm is correct *under its model*. The only bad decision is picking one whose model doesn't match your reality.

Next, Paxos. The algorithm everyone cites, almost nobody implements from scratch, and which is more interesting than its reputation suggests.
