# Paxos, Carefully

Paxos has a reputation. It is supposed to be the algorithm that everyone cites and nobody understands. Leslie Lamport's original 1989 paper, *The Part-Time Parliament*, was rejected on grounds of "excessive cuteness" (it was framed as a parable about a fictional Greek island parliament), sat in a drawer for nine years, and finally appeared in 1998. Even then it took the 2001 follow-up, *Paxos Made Simple*, for most readers to get it.

We're going to treat Paxos with patience. The algorithm is not hard once you untangle the moving parts. We'll split it into its layers, starting with the one-decision core — **Single-Decree Paxos** or **Basic Paxos** — and then building up.

## The single decision

Forget logs for a moment. Suppose we just want a group of processes to agree on *one* value. That is Basic Paxos.

Three roles:

- **Proposers** propose values.
- **Acceptors** vote on proposals.
- **Learners** are told which value was chosen.

In practice every process plays all three roles, but the separation is useful. Roles are not a deployment concern; they are a reasoning tool.

We have `N` acceptors. A **quorum** is any set of `⌈(N+1)/2⌉` acceptors — a strict majority. The safety of Paxos relies on the fact that any two majorities intersect in at least one acceptor.

A **proposal** has two parts: a globally ordered **ballot number** (often called a round or sequence number) and a **value**. Ballot numbers must be totally ordered and unique per proposer, which is easy to arrange — e.g., `(counter, proposer_id)` with lexicographic ordering.

### The two-phase protocol

Paxos runs in two phases.

**Phase 1: Prepare / Promise** — the proposer asks acceptors for permission to propose at some ballot `b`.

```
proposer → acceptors:  PREPARE(b)

acceptor on receiving PREPARE(b):
    if b > any ballot it has previously promised:
        promise not to accept any proposal with ballot < b
        reply PROMISE(b, prior_accepted_b, prior_accepted_value)
    else:
        reject (or silently ignore)
```

A `PROMISE` includes the highest-ballot proposal the acceptor has already accepted (if any). This is the crux of Paxos's safety: an acceptor, when promising for ballot `b`, tells the proposer about any earlier value it is committed to.

**Phase 2: Accept / Accepted** — the proposer, having collected promises from a quorum, chooses a value and asks to accept it.

```
proposer on receiving PROMISE from a quorum:
    if any acceptor reported a prior accepted proposal:
        pick the value from the highest-ballot prior proposal
    else:
        pick any value (e.g., the client's request)
    send ACCEPT(b, chosen_value) to acceptors

acceptor on receiving ACCEPT(b, v):
    if b ≥ highest promised ballot:
        accept(b, v)  -- record it as durably accepted
        reply ACCEPTED(b, v)
    else:
        reject
```

A value is **chosen** when a quorum of acceptors have `ACCEPTED` the same proposal. Learners discover that a value is chosen by observing the acceptor replies (either directly or via a distinguished learner).

That's Basic Paxos. Two round-trips, three roles, a majority quorum, and two invariants:

1. **Promise invariant:** an acceptor that has promised ballot `b` will never accept any proposal with ballot `< b`.
2. **Proposer-chooses-safe-value invariant:** if any acceptor in the quorum reports a prior accepted value, the proposer is forced to propose that same value at the new ballot.

Those two invariants together give you safety: no two distinct values can both be chosen.

### Why the invariants imply safety

Intuitively: suppose `v` was chosen at ballot `b1`. That means some quorum `Q1` accepted `(b1, v)`. Now consider a later ballot `b2 > b1`. For `b2` to succeed, the proposer needed to collect promises from a quorum `Q2`. Since any two quorums intersect, at least one acceptor `a` is in both `Q1` and `Q2`. That acceptor's promise for `b2` included its prior accepted proposal — which was `(b1, v)` or later. The proposer is forced by the second invariant to propose `v` (or a value from an even higher ballot, which, recursively by the same argument, is also `v`).

So once `v` is chosen, every later successful ballot must propose `v`. Safety!

This is the heart of Paxos, and it is worth re-reading until it clicks. The safety argument is purely combinatorial — it doesn't depend on timing, failure detectors, or ordering of messages. It depends only on:

- Quorums intersect.
- Acceptors keep their promises.
- Proposers honor the "pick the highest prior value" rule.

### Why Paxos survives concurrency

Two proposers can try to propose concurrently. They can interfere — proposer A gets promises for ballot 5, then proposer B gets promises for ballot 7 (which invalidates A's promises), then A gets for ballot 9, and so on. Each such interference wastes a round; in pathological adversarial schedules, this can continue forever, producing no decision. That is FLP — precisely the kind of livelock the impossibility result allows.

In practice, we avoid this with a **distinguished proposer** (leader). We'll get to it when we discuss Multi-Paxos.

## Multi-Paxos: chaining decisions

One decision isn't useful; we want a log. The naive approach runs Basic Paxos independently for each log slot. That works — each slot is an instance of Paxos, indexed by slot number. But it's wasteful: every slot pays two round-trips of its own.

**Multi-Paxos** is the optimization: elect a distinguished leader, have the leader run Phase 1 *once* for a range of future slots, and then run Phase 2 repeatedly for each command. The leader is the stable proposer; clients send their commands to the leader; the leader assigns slot numbers and runs the Phase 2 step for each.

This reduces the steady-state cost to **one round-trip per decision** — the leader sends `ACCEPT` to the quorum, the quorum replies `ACCEPTED`, the leader commits. Phase 1 is only re-run when the leader changes.

```
steady state (leader L, followers F1..Fn):

    client → L:            REQUEST(cmd)
    L → F1..Fn:            ACCEPT(ballot, slot, cmd)
    F1..Fn → L:            ACCEPTED(ballot, slot)
    L → client, followers: DECIDE(slot, cmd)
```

The catch, of course, is leader election itself. How do we agree on who the leader is? With a failure detector and... Basic Paxos on a special "leadership" register, or with a lease protocol, or with an external coordination service. In practice this is where most of the subtle bugs hide.

## Fast Paxos

Lamport's 2005 **Fast Paxos** trims the steady-state case to a single round-trip from the *client*, bypassing the leader for the common case. It does this by letting clients broadcast proposals to acceptors directly, at the cost of needing a *larger* quorum (3/4 of acceptors rather than majority) for the fast path. If there's contention (two clients try to propose different values concurrently), the system falls back to classical Paxos.

Fast Paxos is clever and rarely implemented. Most Multi-Paxos deployments stick with leader-based for its simplicity. We mention it so you recognize the name.

## Paxos as Lamport originally described it

The 1989 paper, *The Part-Time Parliament*, tells the story of the Greek island of Paxos, whose parliament passed decrees despite legislators wandering in and out for lunch. The math is encoded as parliamentary procedure — ballots, decrees, a Legislature Register.

The parable is charming and was *completely ineffective as pedagogy*. Nobody could figure out what the actual algorithm was. Lamport himself said in a retrospective: "I thought, and still think, that Paxos is an important algorithm. Since 1980, I had found that the best way to present algorithms clearly was with mathematics. Since math is a universal language..." — missing, as he admits, that most readers were looking for code, not math.

Nine years later, in *Paxos Made Simple*, he presented it as we have above: proposers, acceptors, learners, two phases. This version clicked. But the damage was done — a generation of engineers had concluded that Paxos was impenetrable. Raft was born as a direct response. We'll see it in the next chapter.

## Paxos as it actually gets implemented

The gap between *Paxos Made Simple* and a running implementation is wide. Here are some of the things Paxos doesn't tell you directly:

- **Log compaction.** The log grows without bound as decisions accumulate. In practice you take periodic snapshots of the state machine and discard log prefixes older than the snapshot. This is a non-trivial engineering exercise — you have to handle in-flight requests, rejoining nodes that need to catch up, and recovering from a restart mid-compaction.
- **Membership changes.** Adding or removing a node requires care. The Paxos paper's "α" extension (reconfiguring which acceptors vote on future slots) is correct, but the implementation has pitfalls. Raft devotes a whole section to joint consensus for this reason.
- **Failure detection.** Who notices when the leader dies? Usually a timeout, but the leader's lease, the election protocol, and the followers' clocks all have to agree roughly enough to avoid flapping.
- **Catch-up.** A follower returning from a crash needs to learn all decisions it missed. This is a streaming protocol of its own, with flow control and snapshot-versus-log decisions.
- **Batching and pipelining.** A straightforward Multi-Paxos processes one slot at a time, end-to-end. For throughput, you batch multiple commands into one Accept and/or pipeline consecutive Accepts without waiting for previous ones to commit. Both optimizations are correct under Paxos's invariants but add implementation complexity.

These are not flaws in Paxos — they are engineering problems that every consensus implementation faces. But the original paper hand-waves most of them, which is a significant source of the "Paxos is hard" reputation.

## Quorum arithmetic

Sizing is straightforward for crash-fault tolerance:

- Total acceptors: `N`
- Max simultaneous failures tolerated: `f`
- Quorum size: `Q = f + 1`
- Requirement for safety: any two quorums intersect — `2Q > N` → `N ≥ 2f + 1`

So:

| Nodes `N` | Max failures `f` | Quorum `Q` |
|---|---|---|
| 3 | 1 | 2 |
| 5 | 2 | 3 |
| 7 | 3 | 4 |

Five nodes tolerating two failures is the sweet spot for most production systems. Three nodes is common but tolerates only one failure — awkward during maintenance, since routine reboots look like failures.

Even node counts are pointless: four nodes still only tolerate one failure, but require a quorum of three, costing you more latency and acks than three nodes. Always pick an odd count for the voting membership.

## Correctness intuitions, not formal proofs

If you want the formal correctness argument, Lamport has written it several times; we recommend *Paxos Made Simple* followed by *The Part-Time Parliament* as a readable pair. Here is the intuition at speed:

- **Safety (agreement):** any two decided values at the same slot must be equal. This holds because of the quorum intersection argument above: once a value is decided, the quorum overlap forces every future successful ballot to re-propose that value.
- **Safety (validity):** if a value is decided, it was proposed by some proposer. This is immediate from the protocol — acceptors only accept values they receive in an Accept message.
- **Liveness (termination):** under partial synchrony with a stable leader, every proposal eventually completes. This requires the leader not to be preempted continuously by competing proposers; hence the emphasis on a single stable leader in Multi-Paxos.

## What Paxos teaches

Paxos is not just an algorithm; it is a set of ideas that every later algorithm inherits:

- **Quorum intersection as the bedrock of safety.** Once you see this pattern, you'll notice it everywhere — Raft, PBFT, HotStuff, and beyond.
- **Ballot numbers (or views, or terms, or epochs) as a logical clock for leadership.** The whole "who is in charge" question, which is hellish in distributed systems, is corralled into a single monotonically increasing integer.
- **Separation of agreement from log ordering.** Multi-Paxos shows how to reduce a sequence of agreements to a single leader's assignment of slots, with Phase 1 amortized over many decisions.

Raft, which we meet next, is not a different algorithm so much as a *different presentation* of very similar ideas, engineered for clarity.

## A small Python sketch

Here is a minimal single-decree Paxos, stripped of network and failure handling, just to show the shape. Do not put this in production; it is illustrative.

```python
class Acceptor:
    def __init__(self):
        self.promised_ballot = None   # highest ballot we've promised
        self.accepted_ballot = None   # ballot of most recent acceptance
        self.accepted_value = None

    def prepare(self, ballot):
        if self.promised_ballot is None or ballot > self.promised_ballot:
            self.promised_ballot = ballot
            return ("promise", self.accepted_ballot, self.accepted_value)
        return ("reject",)

    def accept(self, ballot, value):
        if self.promised_ballot is None or ballot >= self.promised_ballot:
            self.promised_ballot = ballot
            self.accepted_ballot = ballot
            self.accepted_value = value
            return ("accepted",)
        return ("reject",)


def propose(proposer_ballot, client_value, acceptors):
    # Phase 1
    promises = []
    for a in acceptors:
        reply = a.prepare(proposer_ballot)
        if reply[0] == "promise":
            promises.append(reply)
    if len(promises) * 2 <= len(acceptors):   # no majority
        return None

    # Choose value: highest prior accepted, or our own
    prior = [(b, v) for (_, b, v) in promises if b is not None]
    if prior:
        _, chosen = max(prior)
    else:
        chosen = client_value

    # Phase 2
    accepts = 0
    for a in acceptors:
        reply = a.accept(proposer_ballot, chosen)
        if reply[0] == "accepted":
            accepts += 1
    if accepts * 2 > len(acceptors):
        return chosen           # decided
    return None
```

This isn't runnable-distributed — there's no network, no concurrency, no crashes — but it is a faithful single-decree sketch. If you read it twice, the invariants should start to feel inevitable rather than mysterious.

## What's next

We've seen consensus the way Lamport saw it: elegant, symmetric, stated in terms of ballots and quorums. Next chapter: the same problem, restated in a way that made a generation of engineers comfortable shipping it. Raft.
