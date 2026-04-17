# Randomized Consensus

So far we have seen two ways to live with FLP. The first: assume partial synchrony, and you get safety always, liveness under well-behaved networks. Paxos, Raft, VR, PBFT, HotStuff all choose this path.

There is another path. **Randomization.**

Michael Ben-Or published *"Another Advantage of Free Choice: Completely Asynchronous Agreement Protocols"* in 1983. The title is a joke with an edge. FLP (1985, but circulating as a draft) had just shown that deterministic asynchronous consensus was impossible. Ben-Or's response: *fine, then let's not be deterministic*. Let each process flip a coin when it can't decide. It turns out that is enough.

Randomized consensus sidesteps FLP entirely. It is safe *and* live (with probability 1) in pure asynchrony. It does not need partial synchrony, timeouts, or leaders. The price is probabilistic — under worst-case adversarial scheduling, expected number of rounds can be exponential in `n` for the simplest variants. Modern variants dramatically improve this.

You will rarely see pure randomized consensus deployed standalone in mainstream systems. Its ideas, however, are baked into several modern BFT designs, and understanding it opens your conceptual toolkit.

## Ben-Or's algorithm

The setting: `N` processes, up to `f` Byzantine, synchronous message passing in rounds (each round, every process sends messages, receives messages, decides). But note — we can't *guarantee* the network is synchronous; we use the round structure as a conceptual device. Ben-Or's algorithm still works in asynchrony; rounds are just ways to group messages.

Each process starts with a bit `v ∈ {0, 1}` and wants to agree on some single bit. The algorithm runs in phases, each phase has two rounds:

```
phase k, round 1:
    broadcast <PROPOSE, k, v_i>
    wait for N-f <PROPOSE, k, *> messages
    if > N/2 of them have the same value w:
        broadcast <VOTE, k, w>
    else:
        broadcast <VOTE, k, ⊥>    // "no majority"

phase k, round 2:
    wait for N-f <VOTE, k, *> messages
    if >= f+1 of them are <VOTE, k, w> with w ≠ ⊥:
        decide w (and keep sending VOTE for this w)
    else if exists one <VOTE, k, w> with w ≠ ⊥:
        v_i := w
    else:
        v_i := flip_coin()       // random bit
```

The algorithm's skeleton:

1. Propose what you currently believe.
2. If most of your peers agree, you vote for that value.
3. Otherwise, if nobody in your received quorum reached agreement, you **flip a coin** and pick a new value for the next phase.

### Why it terminates

The key insight: if *all* processes happen to flip to the same coin value in the same phase, the protocol decides in the next round. Each process independently has a 50% chance of flipping 0 vs. 1; the chance of all `N` landing the same way in a given phase is `2 · (1/2)^N = 2^{1-N}` for 2 values. So expected rounds to termination is exponential in `N`.

That's bad for `N = 100`. It's tolerable for `N = 10`.

### Why it's safe

A value can only be decided in round 2 of some phase. For a decision, the decider saw `f+1` `VOTE`s for some value `w`. These include at least one honest voter. An honest voter votes for `w` only if it saw a majority of `PROPOSE`s for `w` in round 1. Two different decisions in the same phase would require two disjoint majorities of `PROPOSE`s — impossible. Decisions in subsequent phases are forced to also be `w` because, once `f+1` `VOTE`s for `w` are out, any process in round 2 of the next phase sees at least one `VOTE` for `w` (by quorum intersection) and either decides `w` or adopts it as its next `v_i`.

Safety does not depend on the coin being fair or even random — it holds no matter what the coin does. Randomness only helps liveness.

## Common coins: smarter randomness

The exponential expected rounds of Ben-Or is a problem. The fix: instead of each process flipping its own coin, have all processes share a single coin, unpredictable to the adversary but consistent across processes. If all honest processes see the same bit, and that bit was chosen randomly from the adversary's perspective, a decision happens in one round with probability 1/2 — expected rounds become constant, not exponential.

Shared-coin protocols require cryptographic primitives. The common approach, due to Rabin (1983) and Cachin, Kursawe, and Shoup (2000), uses **threshold cryptography**:

- There is a secret key split into shares, one per process.
- To reveal the round's coin, at least `f+1` processes must contribute their share.
- Until `f+1` shares are collected, the coin value is unknown to anyone — including an adversary who controls up to `f` of the processes.

A process participating in a round commits to its share; the adversary cannot delay the coin reveal without blocking the whole group; when `f+1` shares arrive, the coin reveals deterministically to everyone.

Now each round decides with probability 1/2 (or higher). Expected rounds to termination: 2. Practical randomized BFT becomes viable.

## Modern randomized BFT

HoneyBadgerBFT (Miller et al., 2016) is the emblematic modern asynchronous randomized BFT protocol. Its design:

- Each process proposes a batch of transactions.
- A **Reliable Broadcast (RBC)** primitive ensures that every honest process eventually gets every other process's proposal.
- A **Binary Byzantine Agreement (BBA)** — using Ben-Or-style randomized consensus with a common coin — runs in parallel for each process's batch, deciding whether to include it.
- Batches that pass BBA are combined (in a canonical order) into the final block.

HoneyBadgerBFT is the first BFT protocol practical at Internet scale *and* asynchronous-safe. It does not rely on any timing assumptions for safety or liveness. Its throughput depends on message and cryptographic overhead; early benchmarks showed tens to hundreds of transactions per second per node in WAN deployments, which, for an asynchronous BFT protocol, was a breakthrough.

Successors include **Dumbo** and **DispersedLedger** (both improvements on HoneyBadger's latency), and **Speeding Dumbo** (further optimizations). These papers are active research as of 2026, and the community is converging on designs that are actually practical asynchronous BFT.

## Where randomization shows up in mainstream BFT

Even outside of explicit "randomized BFT" papers, randomness appears subtly in production protocols:

- **Random leader selection** in some HotStuff variants (to prevent adversarial scheduling of leader slots).
- **Random tiebreaking** in vote collection (to avoid deterministic tie patterns that an adversary could exploit).
- **Randomized timeouts** in Raft and PBFT (to break livelock among simultaneous candidates).

None of these rise to "the algorithm is randomized" in Ben-Or's sense. But they borrow the key insight: an adversary is weaker when you insert genuine randomness into the protocol.

## Why randomized consensus is rare in production

Given that it's theoretically lovely — safe and live without timing assumptions — why isn't randomized consensus the default?

Several reasons:

- **Engineering cost.** Threshold cryptography is complex, requires distributed key generation, and adds cryptographic overhead. For most deployments, "partial synchrony + occasional view change" is cheaper.
- **Operator intuition.** Random algorithms are harder to debug. "Why did this view take 14 rounds to terminate?" — because the coin flips landed against you. That explanation is correct but unsatisfying.
- **Network reality.** Real networks are partially synchronous the vast majority of the time. You pay for the optimization (asynchronous safety) but rarely cash it in. Deterministic protocols with timeouts are simpler and empirically adequate.
- **Throughput.** Until recently, randomized BFT was orders of magnitude slower than PBFT/HotStuff in the common case. Modern designs are closing this gap, but classical protocols had a long head start.

It's a case of the theoretically preferable solution losing to the empirically good enough one — a common pattern in distributed systems.

## What randomized consensus teaches

- **FLP isn't load-bearing in practice.** You can route around it with randomness just as easily as with timing assumptions. Knowing both routes is clarifying.
- **Coins change the adversary's power.** A deterministic protocol lets the adversary schedule messages to keep the protocol undecided; a randomized protocol denies the adversary that power (if the coin is truly unpredictable).
- **Threshold cryptography is a gift.** It makes "shared randomness" a concrete, implementable primitive. It also underlies HotStuff's QC aggregation — the same tool, used for a different purpose.
- **Asynchronous safety is a real, valuable property.** Systems that are correct in pure asynchrony (not just partial synchrony) fail gracefully under extreme conditions where partially-synchronous systems do not.

## A note on proofs and practice

Randomized consensus proofs are delicate. *"Terminates with probability 1"* is not the same as *"terminates in constant rounds."* Some protocols terminate in expected constant rounds but have fat tails (high variance). Others terminate in expected-constant *and* bounded-variance rounds. If you read a randomized BFT paper, look for the probability claim *and* the bound on the number of rounds for termination with high probability — those are different guarantees, and the paper may be claiming just one.

The textbook treatment lives in Cachin, Guerraoui, and Rodrigues, *Introduction to Reliable and Secure Distributed Programming* (2nd ed., 2011), Chapter 5. It's the clearest exposition I know.

## Closing the consensus-algorithm tour

With randomized consensus, we've now seen the major branches of the family tree:

- **Crash-fault tolerant, deterministic, leader-based.** Paxos, Raft, VR.
- **Byzantine-fault tolerant, deterministic, leader-based, quadratic.** PBFT.
- **Byzantine-fault tolerant, deterministic, rotating-leader, linear.** HotStuff and descendants.
- **Byzantine-fault tolerant, randomized, leaderless.** Ben-Or; HoneyBadgerBFT and successors.

Each is the right answer to a particular set of assumptions. No algorithm dominates on all dimensions. The rest of the book turns to practice — what these algorithms look like in systems you actually run.
