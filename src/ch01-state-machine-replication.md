# The Problem: State Machine Replication

Before we can talk about consensus algorithms, we need to be precise about the problem they solve. Vague problems invite vague solutions; precise problems invite good ones.

The word *consensus* gets overloaded. Sometimes it means "agreement" in an abstract political sense. Sometimes it means "the particular value a group of processes settle on." Sometimes it means "the protocol those processes run." All three meanings appear in the literature, sometimes in the same paper. We will use it in the technical sense: a group of processes, each starting with an input, must all decide on a single output, such that:

1. **Agreement** — no two correct processes decide different values.
2. **Validity** — the decided value was proposed by at least one process.
3. **Termination** — every correct process eventually decides.

This is the definition Lamport and others have refined over decades. Any protocol that satisfies these three is a consensus protocol. The hard part is satisfying all three at once under realistic assumptions about failures and timing.

But consensus as defined above is a one-shot problem. You propose, you agree, you decide — once. Real systems want to agree on a *sequence* of values. That is where state machine replication comes in.

## The canonical use case

Suppose you have a service — a database, a configuration store, a lock manager, whatever. The service is a deterministic state machine: given the current state and an input command, it produces a new state and an output. Single-node deterministic, and trivial.

Now you want this service to survive node failures without downtime or data loss. So you run it on several nodes. The challenge: every replica has to end up in the same state. If one replica thinks the balance is `$100` and another thinks it is `$200`, we have a problem that no amount of retry logic will fix.

The classical trick, due to Leslie Lamport in the late 1970s and refined by Fred Schneider in a 1990 survey paper, is:

> **State Machine Replication (SMR):** If every replica starts in the same initial state and applies the *same* sequence of deterministic commands in the *same* order, every replica ends up in the same state.

That is, the problem of "keep replicas in sync" reduces to "agree on a totally-ordered log of commands." The state machine is trivially deterministic; the replication is entirely in the log.

```
clients ──▶ consensus protocol ──▶ ordered log ──▶ each replica applies
                                                    commands in order

  ┌───────┐           ┌───────────┐
  │ put x │─────┐     │ 1: put x  │───▶ replica A: apply 1,2,3 ── state S
  │ put y │─────┼────▶│ 2: put y  │───▶ replica B: apply 1,2,3 ── state S
  │ cas z │─────┘     │ 3: cas z  │───▶ replica C: apply 1,2,3 ── state S
  └───────┘           └───────────┘
```

Every consensus algorithm in this book — Paxos, Raft, PBFT, HotStuff, all of it — is, at its heart, a mechanism for producing that totally-ordered log of commands despite failures and asynchrony. The service on top of the log is an implementation detail. The log is the point.

This decomposition is worth internalizing:

- **Below the log**: consensus. Hard. Full of tradeoffs. The subject of this book.
- **Above the log**: deterministic state machine. Easy. Just a function.

When people say "we built our own Raft" they almost always mean they built the below-the-log part. The above-the-log part is usually whatever they already had.

## What "same state" actually means

"All replicas end up in the same state" sounds simple until you pick at it. Same state *when*? What if one replica has applied commands 1–100 and another has applied 1–95? Are they in the same state?

The standard move is to say: replicas are *eventually* in the same state for any prefix of the log they have both applied. Specifically:

- All correct replicas apply the same totally-ordered sequence of commands.
- A replica that has applied commands `1..k` is in the state that results from applying `1..k` to the initial state.
- Replicas may lag — replica B may be at 95 while A is at 100 — but they never *disagree* on commands they have both applied.

This is sometimes called **linearizability of the log**, or when you push the guarantee up through the state machine, **linearizability of the service** (Herlihy and Wing, 1990). Linearizability means, roughly, that even though operations are happening concurrently across many clients, the system behaves as if each operation took effect instantaneously at some point between its invocation and response.

Linearizability is the strongest consistency model in common use. It is also not the only useful one — there's sequential consistency, causal consistency, eventual consistency, and more. We'll get to the tradeoffs in Chapter 11. For now, hold on to:

> Consensus produces an ordered log. An ordered log makes replicas applying deterministic commands indistinguishable from a single machine.

## Not every system needs this

State machine replication is powerful and expensive. Every write has to go through consensus, which means every write pays for multiple rounds of network I/O. If all you need is a cache, or a log of events where duplicate or out-of-order delivery is tolerable, or a search index that can lose a few documents without ruining anyone's day — SMR is overkill. Eventually-consistent systems (Dynamo-style) make a different tradeoff: they accept staleness and sometimes conflicts in exchange for availability under partitions and much higher write throughput.

We'll meet those systems later. For now, assume we are building the strict systems — the ones where order matters and correctness beats throughput. The kind of systems whose bugs get called incidents.

## Why this isn't the blockchain problem

Blockchain is also in the business of producing an ordered log — the chain is nothing but a sequence of blocks, each containing an ordered list of transactions. So why do blockchains look nothing like Raft?

Because they solve a different problem:

| Dimension | Classical SMR | Public blockchain |
|---|---|---|
| Membership | Fixed, known roster | Open; anyone can join |
| Identity | Authenticated (keys, certs) | Pseudonymous |
| Failure model | Crash or Byzantine, bounded fraction | Byzantine, Sybil-prone |
| Network | Private or semi-trusted | Public internet |
| Incentives | Operators are paid to run nodes | Participants must be economically motivated |
| Finality | Usually within milliseconds | Probabilistic, or eventual after minutes |

Every classical consensus algorithm assumes you know who is in the room — that you can enumerate the participants, authenticate their messages, and count them. The moment you can do that, you can have a `3f+1` or `2f+1` quorum. You can run leader-based protocols without worrying about Sybil attacks. You can finalize decisions in one network round-trip.

Blockchains cannot assume that. A public blockchain is trying to bootstrap agreement among a dynamic, pseudonymous, adversarial crowd. That's a fundamentally harder problem, and it needs expensive tools — proof-of-work, proof-of-stake, economic incentives, probabilistic finality.

Different problem, different tools. Neither is "the" consensus problem; they sit at different points on the tradeoff surface.

## A note on terminology

A few terms we will use throughout the book, defined once so we are consistent:

- **Process / node / replica / server** — used interchangeably to mean one member of the consensus group. When distinction matters we'll say so.
- **Client** — the external caller submitting commands to the service on top of the log.
- **Command / operation / request** — the thing submitted by a client, destined for the state machine.
- **Proposal** — a candidate value (often, but not always, a command) that a consensus instance is trying to decide on.
- **Decision / commit** — the moment a value becomes durable and cannot be reversed.
- **Quorum** — a subset of processes large enough that any two quorums intersect in at least one correct process. We will derive the sizes in later chapters.
- **Leader / primary / proposer** — the process currently driving decisions. Some protocols have one; some rotate; some have none.
- **View / epoch / term** — a logical time period during which a particular leader (or configuration) is in charge. When the leader changes, the view changes.

Those last three words — view, epoch, term — are the same idea wearing different hats in different papers. Paxos calls it a *ballot number*; Raft calls it a *term*; Viewstamped Replication and PBFT call it a *view*; some papers say *epoch*. We'll use the native term for each algorithm when we are inside its chapter.

## What you should take away

- Consensus in the technical sense means: agreement, validity, termination.
- Real systems want a *log* of decisions, and they get it by running consensus repeatedly — that is what we call **state machine replication**.
- SMR converts the hard problem ("keep replicas in sync") into a pair of easier ones: "agree on an ordered log" (consensus) plus "apply commands deterministically" (state machine).
- Blockchains also produce an ordered log, but under different assumptions — open membership, pseudonymous identity, Byzantine networks — so their solutions look different.
- The rest of this book is about the "agree on an ordered log" half, under the assumption of known, authenticated participants.

Next up, the constraints: what the theorists proved we can't do, and what the escape hatches look like.
