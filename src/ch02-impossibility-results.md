# The Impossibility Results

Before we get to the algorithms, we have to know what the rules of the game forbid. Two results above all define the shape of the field:

- **FLP** — Fischer, Lynch, and Paterson (1985). "Impossibility of Distributed Consensus with One Faulty Process."
- **CAP** — Brewer (2000), Gilbert and Lynch (2002). "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services."

If you have worked in distributed systems for a while, you have seen these cited as banners waved to end arguments. That is a shame, because both are more interesting and more nuanced than the banner versions suggest. We'll unpack each on its own terms, say what it *actually* rules out, and describe the escape hatches real algorithms use.

## FLP: asynchronous consensus is impossible

Let's start with what FLP says.

**Claim:** In a fully asynchronous system with even *one* process that might crash, there is no deterministic protocol that solves consensus.

"Fully asynchronous" is crucial. In FLP's model:

- There is no upper bound on message delay — any finite delay is possible.
- There is no upper bound on processing speed — any finite time to compute is possible.
- Processes have no clocks.
- The adversary scheduling messages and processing steps is arbitrary (but finite).

In that world, FLP proved you cannot satisfy all three of agreement, validity, and termination, deterministically, if any process might fail by crashing.

**The intuition**, stripped of formalism:

Imagine a protocol that purports to solve consensus. At any point, the protocol is in some *configuration* — the state of every process plus all in-flight messages. Configurations can be classified:

- **0-valent**: every run from here decides 0.
- **1-valent**: every run from here decides 1.
- **bivalent**: some runs from here decide 0, some decide 1.

Initial configurations must include at least one bivalent one (otherwise the decision is fixed by the inputs and not really "agreement"). FLP's elegant trick is to show that from any bivalent configuration, the adversary can choose a message delivery order that leads to another bivalent configuration — *forever*. The protocol can be kept in a bivalent state indefinitely by carefully timing one message at a time. So it never terminates.

Notice what the adversary is doing. It isn't crashing processes arbitrarily (it only gets one crash). It's *scheduling messages* to keep the protocol from making up its mind. This is the heart of the result: asynchrony plus even a single crash gives an adversary enough power to prevent termination.

**What FLP does NOT say.** This is where confusion reigns. FLP does not say consensus is impossible in practice. It does not say you shouldn't bother. What it says is precise:

- In the *fully asynchronous* model,
- for a *deterministic* protocol,
- tolerating *even one* crash,
- you cannot guarantee *termination* in every run.

Each italicized word is a relaxation that unlocks possibility.

- **Relax "fully asynchronous":** assume the network is *eventually* synchronous — that after some unknown time, messages are delivered within some bound. This is the **partial synchrony** model of Dwork, Lynch, and Stockmeyer (1988). Real networks are partially synchronous most of the time. All practical consensus algorithms assume partial synchrony for *liveness*, while remaining safe in full asynchrony.
- **Relax "deterministic":** use randomization. If each process can flip a coin, you can break the adversary's scheduling power. Ben-Or's 1983 algorithm does exactly this, at the cost of probabilistic termination. Chapter 10 is about this family.
- **Relax "tolerating a crash":** if no processes fail, consensus is easy. The scary part of FLP is that even one failure ruins deterministic asynchronous agreement.
- **Relax "termination":** if you give up liveness, safety alone is easy — just never decide. Obviously not useful, but it shows where the impossibility bites.

Practical algorithms combine these relaxations in predictable ways:

- Paxos, Raft, VR, PBFT, HotStuff are all **safe under asynchrony, live under partial synchrony.** They will never produce conflicting decisions no matter how adversarial the network. They will make progress when the network is "well-behaved enough."
- Randomized protocols (Ben-Or, and modern descendants like HoneyBadger-style common-coin protocols) are **safe always, live with probability 1** — meaning they terminate with probability approaching 1 over time, even in full asynchrony.

This is the FLP escape hatch every practical system takes. **Safety always, liveness under timing assumptions (or in probability).** Keep that phrase in your pocket; you'll see it repeatedly.

## CAP: pick two

CAP is the most cited, most misquoted, most vehemently argued-about theorem in distributed systems. It was informally stated by Eric Brewer in a 2000 PODC keynote and formally proven by Seth Gilbert and Nancy Lynch in 2002. Let's do it correctly.

**Claim:** A networked shared-data system cannot simultaneously provide *Consistency*, *Availability*, and *Partition tolerance*.

Now the words:

- **Consistency (C)**, in CAP, means **linearizability** — the strict ordering we discussed in Chapter 1. (Note: the C in CAP is *not* the C in ACID. The C in ACID is "constraints preserved." Same letter, different concepts. Blame computer science.)
- **Availability (A)** means every request to a non-failed node receives a non-error response — not an error, not a timeout, a real response with data.
- **Partition tolerance (P)** means the system continues to operate despite an arbitrary number of messages being dropped between nodes.

The theorem says you can't have all three at once when a partition happens.

### Why people draw the triangle wrong

A thousand slide decks present CAP as "pick two out of three" and draw a triangle with arrows suggesting you choose two corners: CA, CP, or AP. This framing is essentially wrong.

**Partitions are not optional.** Networks drop messages. Switches fail. Cables get unplugged. Cloud providers have bad days. A system that doesn't tolerate partitions doesn't *exist*; it just fails badly when partitions inevitably happen. So P is not a dial you choose — it's reality.

What CAP really says is: **when a partition happens**, you have to choose between C and A. Either you refuse to serve some requests (sacrifice availability) so that clients on different sides of the partition don't see conflicting states (preserve consistency); or you keep serving (preserve availability) and accept that different sides will diverge (sacrifice consistency).

In the no-partition case, you can absolutely have both consistency and availability. That's what you build a system for. CAP is a statement about degraded operation, not a design matrix.

### CP and AP systems

With that cleaned up:

- **CP systems** sacrifice availability during partitions. They refuse writes (and maybe reads) to minority partitions. etcd, Consul, ZooKeeper, Spanner, most consensus-based stores — CP.
- **AP systems** sacrifice consistency during partitions. They keep serving on both sides and reconcile later. Dynamo, Cassandra, Riak (classic Dynamo-style) — AP.

Nearly every consensus algorithm in this book sits firmly on the CP side. That's by design: SMR wants strict ordering, which means some node has to be authoritatively "current," and a minority partition can't be sure it still is.

### PACELC: the more honest theorem

Daniel Abadi proposed **PACELC** in 2010 as a refinement: if there is a **P**artition, choose between **A**vailability and **C**onsistency; **E**lse (no partition), choose between **L**atency and **C**onsistency.

This matters because consensus algorithms trade latency for consistency *even when the network is fine.* Every write has to go through a quorum; that's at least one round-trip to `f+1` or `2f+1` peers. If you want single-digit-millisecond tail latencies and you have cross-region replicas, consensus will cost you. The "no partition" choice is not free.

PACELC is less famous than CAP but more useful for reasoning about real systems. When someone asks "should we use Raft?" the honest answer often starts with "how much latency are you willing to pay per write?"

## What these results actually rule out, and what they don't

FLP and CAP together forbid:

- A deterministic protocol that, under fully asynchronous networks with at least one possible crash, always terminates.
- A system that, during a partition, keeps all nodes responsive *and* strictly consistent.

They do **not** forbid:

- A deterministic protocol that is always safe and terminates under partial synchrony. (Paxos, Raft, PBFT, HotStuff.)
- A system that is consistent when the network is healthy and unavailable only during partitions. (etcd, Consul, ZooKeeper.)
- A system that is always available and converges after partitions heal. (Cassandra, DynamoDB in default modes.)
- A randomized protocol that terminates with probability 1 under asynchrony. (Ben-Or family.)

So the results are not obstacles to building systems; they are shape constraints on the systems you can build. Every algorithm in this book hugs one of the escape hatches.

## The practical escape hatches, summarized

Here's a compact table you can carry:

| Escape hatch | What it gets you | What it costs |
|---|---|---|
| Partial synchrony | Deterministic liveness when network is "normal." | Liveness pauses during bad periods (leader elections, retries). |
| Randomization | Probabilistic liveness without timing assumptions. | Extra message complexity; harder to reason about; rarely used alone in production. |
| Weaker consistency | Always available, high throughput. | You have to handle conflicts in application code. |
| Unanimous quorum | Simplicity. | Can't make progress if *any* node is down. |
| Hierarchical designs | Limits the scope of consensus. | More moving parts; harder to operate. |
| Clock synchronization (Spanner) | Faster reads, external consistency. | Requires tightly engineered clock infrastructure (TrueTime). |

The rest of this book is a study of how each algorithm chooses among these.

## A philosophical aside

FLP and CAP are often taught like ghost stories — "the distributed systems gods forbid certain things." That framing makes them seem bigger than they are. Both are simply **statements about model-level tradeoffs under adversarial conditions.** They are useful precisely because they tell you which corners of the design space are empty, so you don't waste time looking there.

The analog in classical physics would be conservation of energy. Nobody rants about conservation of energy on Twitter; they just use it to rule out perpetual motion machines and move on. FLP and CAP deserve the same treatment. They rule out a few specific kinds of perpetual motion — and then they get out of your way.

With the impossibility landscape mapped, we can turn to the next piece of the puzzle: the kinds of failures you actually need your algorithm to tolerate.
