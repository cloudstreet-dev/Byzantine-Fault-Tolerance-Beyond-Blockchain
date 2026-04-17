# HotStuff and the Linear BFT Family

PBFT changed the world by making Byzantine consensus fast enough to use. HotStuff changed it again by making Byzantine consensus scalable enough to use with *many* replicas.

Maofan Yin, Dahlia Malkhi, Michael Reiter, Guy Gueta, and Ittai Abraham published *HotStuff: BFT Consensus in the Lens of Blockchain* at PODC 2019. The paper's name is a tell — it was born out of the realization that classical BFT (PBFT with its `O(n²)` communication) could not keep up with the `n` that blockchain consortia wanted.

HotStuff's two contributions:

1. **Linear communication per decision.** By having replicas send votes to the leader rather than gossiping with each other, and aggregating those votes with threshold signatures, HotStuff cuts the per-decision message count from `O(n²)` to `O(n)`.
2. **Optimistic responsiveness.** In partial synchrony, HotStuff progresses as fast as the network allows, without waiting for fixed timeouts in the common case — a property PBFT lacks.

## The setting

- `N = 3f + 1` replicas, as in PBFT.
- Up to `f` Byzantine.
- Digital signatures (actually threshold signatures — more on this shortly).
- A rotating leader — each view's leader is different. Unlike PBFT, where the leader stays until suspected faulty, HotStuff rotates the leader every consensus round.
- Messages carry **Quorum Certificates (QCs)** — threshold-signature aggregations of `2f+1` matching votes.

## Quorum certificates and threshold signatures

Core HotStuff move. Instead of every replica broadcasting `PREPARE` to every other replica (quadratic), every replica sends its signed vote to *the leader*. The leader collects `2f+1` votes and aggregates them into a **Quorum Certificate** — a single, constant-size object that is verifiable proof that `2f+1` replicas voted for the same thing.

Threshold signatures make this possible. A `(t, n)`-threshold signature scheme lets any `t` of `n` signers produce a single signature on a message, verifiable with a single public key. The leader, on collecting `t = 2f+1` signature shares, combines them into one threshold signature. The resulting QC is one object, one signature-verify away from proof.

Without threshold signatures, a QC would be `2f+1` separate signed votes — `O(n)` in size, and `O(n²)` to broadcast. With threshold signatures, a QC is `O(1)` in size, and broadcast is `O(n)`.

BLS signatures (Boneh-Lynn-Shacham) are the standard choice — their aggregation is particularly clean. Cryptographic cost is nontrivial but amortizable.

## The basic HotStuff protocol

Basic HotStuff (as opposed to Chained HotStuff) runs in four phases per decision. Each phase is a round-trip between leader and replicas, using QCs to carry proof of the previous phase.

```
PREPARE      PRE-COMMIT      COMMIT         DECIDE
   │              │             │              │
   └─leader       └─leader      └─leader       └─replicas execute
    proposes       sends         sends
    block          prepareQC     precommitQC
```

Specifically:

1. **Prepare phase.**
    - Leader proposes a new block `b` that extends the highest `prepareQC` seen.
    - Each replica votes if it safe to vote (details below).
    - Leader collects `2f+1` votes into `prepareQC`.
2. **Pre-commit phase.**
    - Leader broadcasts `prepareQC`.
    - Each replica, upon seeing `prepareQC`, votes for pre-commit.
    - Leader collects `2f+1` votes into `precommitQC`.
3. **Commit phase.**
    - Leader broadcasts `precommitQC`.
    - Each replica votes for commit.
    - Leader collects `2f+1` votes into `commitQC`.
4. **Decide phase.**
    - Leader broadcasts `commitQC`.
    - Each replica, upon seeing `commitQC`, executes the block's commands.

The safety rule for voting is the **locked-QC rule**: a replica votes for a block `b` if `b` extends the block it is locked on (the one with its highest `precommitQC`) or if `b`'s parent's QC has a higher view than the locked one.

The reason for *four* phases rather than PBFT's three is subtle. The extra phase gives HotStuff a property called **optimistic responsiveness** — the protocol advances as fast as the leader can collect votes, without waiting for a fixed timeout, because the additional phase's structure ensures that a new leader can always make progress based on QCs from past views, without waiting for a "view-stable" timer.

Don't worry if you have to re-read the paper for this; it is the most debated part of the design.

## Chained HotStuff

Four phases per decision is a lot of round-trips. Chained HotStuff pipelines them. Each view's single phase of messages serves double duty:

- It is the **decide** phase for some earlier block `b`.
- It is the **commit** phase for a slightly-less-earlier block `b'`.
- It is the **pre-commit** phase for the block after that.
- It is the **prepare** phase for the current block.

One round of message exchange, four blocks worth of progress simultaneously. The net effect: one message round-trip per *decision*, amortized.

```
view v:       prepare(b4) | pre-commit(b3) | commit(b2) | decide(b1)
view v+1:     prepare(b5) | pre-commit(b4) | commit(b3) | decide(b2)
view v+2:     prepare(b6) | pre-commit(b5) | commit(b4) | decide(b3)
```

Each block is decided three views after it is prepared. Latency per decision is higher (measured in views), but *throughput* is one decision per view. For a system processing many requests, throughput is the relevant metric.

## Rotating leaders

In PBFT, the leader stays in place until it is suspected faulty. HotStuff rotates the leader every view, without waiting for suspicion. Why?

- **Fairness.** No single replica has disproportionate influence.
- **Simplicity.** Every view has a clear new leader; view changes and normal operation are unified.
- **Liveness.** If the current leader is Byzantine or slow, the next view automatically gives someone else the chance to lead. No explicit "is the leader faulty?" decision needed.

The cost: you lose the "stick with a good leader" optimization. In the common case where the current leader is correct and fast, HotStuff still rotates. The linear communication pattern compensates for this by making each leader-round cheap.

## View change and liveness

A view change in HotStuff is just "the next view's leader starts proposing." There is no separate view change protocol. If a leader is slow or silent, replicas time out and advance to the next view. The new leader gathers `2f+1` `NEW-VIEW` messages (each containing the sender's locked QC and prepareQC) and proposes a new block extending the highest safe block.

This unified structure is part of why HotStuff is easier to reason about than PBFT — there's no discontinuity between "normal case" and "view change."

## LibraBFT, DiemBFT, AptosBFT

Facebook's Libra project (2019) needed a consensus protocol for its permissioned blockchain with dozens to hundreds of validators. Classical PBFT wouldn't scale; proof-of-stake was overkill for a known validator set. They picked HotStuff and extended it into **LibraBFT**.

Libra was renamed Diem in 2020 (and DiemBFT with it) and shut down in 2022. Meta transferred the technology to the Diem Association, which sold parts of it off. Meanwhile, several ex-Diem engineers founded Aptos, which ships **AptosBFT** (another HotStuff descendant, currently part of the Aptos blockchain network). Sui, another ex-Diem offshoot, uses a different consensus design (Narwhal+Bullshark).

Key extensions these projects made:

- **Pacemaker.** A separate module that handles view timeouts, randomization, and leader selection, decoupled from the core consensus logic. This makes the protocol testable and tunable.
- **Pipelined commit rule.** Three consecutive chained views of the same branch imply commit of the oldest of those three. Concretely: if a block has a QC that is "grandchild-QC'd" by a block at view `v+2`, it commits. Simplifies the safety argument.
- **Reconfiguration.** Validator set changes via epoch boundaries — consensus is run within an epoch, and epoch transitions include a reconfiguration step that atomically swaps validator sets.
- **Execution optimizations.** Batching, parallel execution of independent transactions, careful memory management. Less "consensus" per se and more "making the practical system fast."

These are not fundamental changes to HotStuff's consensus mechanics. They are the additional engineering it takes to ship a production blockchain using HotStuff.

## The tradeoffs

HotStuff vs. PBFT:

| Dimension | PBFT | HotStuff (chained) |
|---|---|---|
| Communication per decision | `O(n²)` | `O(n)` |
| Cryptographic ops per decision | Signatures/MACs across `O(n²)` messages | Threshold signature aggregation, `O(n)` |
| Leader stability | Sticky (until suspected) | Rotates every view |
| Latency (phases per decision) | 3 | 4 (but pipelined → effective 1 per decision) |
| View change complexity | Separate, intricate protocol | Unified with normal case |
| Implementation lines | Large; VR- and PBFT-like state machine | Modular with pacemaker; surprisingly compact |
| Ecosystem | Handful of research and industry impls | Multiple production deployments (Diem/Aptos, others) |

What you give up:

- **Throughput under an optimally-correct leader** is slightly lower in HotStuff, because leader rotation means every view has new cold caches, new leader-overhead costs. A PBFT system with a very reliable primary might outperform HotStuff at small `n`.
- **Simpler to grasp**: PBFT's three phases are easier to internalize the first time than HotStuff's four phases + chaining. HotStuff's elegance is structural but takes a second read.

What you gain:

- **Scalability**. `n` can be 100+ without the quadratic cost dominating.
- **Uniformity**. Normal case and view change share structure.
- **Optimistic responsiveness**. Common-case progress tracks network speed.

## The linear-BFT family

HotStuff was not the first attempt at reducing BFT communication, but it was the one that found the right combination of rotating leaders, QCs, and pipelining. Several other proposals in the same family:

- **SBFT** (Gueta et al., 2019) — uses threshold signatures for vote aggregation; closer to PBFT in structure.
- **Tendermint** (Buchman, 2016) — PBFT-descended, rotating leader, used in Cosmos. Predates HotStuff. Linear in common case, but has a different liveness property.
- **Narwhal and Bullshark** (Mysticeti, Sui) — mempool/consensus separation; DAG-based.
- **Jolteon / DiemBFT v4** — a HotStuff variant trading one phase for an exponential-backoff liveness rule.

The common thread: reduce communication complexity and unify the view change path.

## An observation for practitioners

If your system has fewer than, say, 10 replicas, PBFT is probably fine. The `O(n²)` factor at `n=10` is 100 messages per decision, which is still cheap.

If your system has 20+ replicas — permissioned blockchain territory — you want a linear-BFT variant. HotStuff and its descendants are the default choice.

If your system has more than about 200 replicas, classical BFT starts to stress even linear protocols, and you start to see research into randomized, asynchronous, or DAG-based BFT. That's roughly where we are in 2026.

## What HotStuff teaches

- **Communication complexity is the binding constraint at scale.** Safety alone isn't enough — an algorithm has to be cheap enough to run.
- **Pipelining amortizes latency.** You can have a 4-phase protocol that delivers decisions at a rate of one per phase.
- **Threshold signatures are a structural primitive, not just an optimization.** They change what collective agreement "looks like" on the wire.
- **Unifying normal case and view change** makes implementations tractable.

Next, a stranger branch of the tree: randomized consensus, which sidesteps FLP by flipping coins.
