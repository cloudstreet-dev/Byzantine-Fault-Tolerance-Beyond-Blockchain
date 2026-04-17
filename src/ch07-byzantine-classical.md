# Byzantine Fault Tolerance, Classical

We have spent six chapters assuming that nodes, when they fail, fail cleanly. They stop. They don't send any more messages. When they come back, they come back honestly with a consistent view of their own past.

Now drop that assumption. What if a node can do *anything*?

Lie about its log. Send different votes to different peers. Forge messages, if the cryptography allows. Collude with other faulty nodes. Act perfectly normal for an hour and then betray.

This is the **Byzantine failure model**, named by Leslie Lamport, Robert Shostak, and Marshall Pease in their 1982 paper *"The Byzantine Generals Problem."* The paper frames the model as a story about Byzantine generals besieging a city, some of whom might be traitors. The story is charming; the math under it is unforgiving.

## The generals' problem

Several generals, each commanding a division outside the city, must agree on a single plan: attack or retreat. They communicate by messenger. Some of the generals may be traitors who will try to prevent the loyal ones from reaching agreement. Traitors can:

- Send different plans to different generals.
- Lie about what other generals told them.
- Collaborate with other traitors.

Can the loyal generals agree on a plan, and can they carry out that plan together?

Translated to distributed systems: can correct nodes reach consensus despite some peers behaving arbitrarily?

## The core theorem

The paper's main result: **`3f + 1` nodes are necessary to tolerate `f` Byzantine failures.**

This is tight — you cannot do it with fewer. The bound applies under a particular communication model (**unauthenticated** messages — no cryptographic signatures). With signatures, the bound changes, which we'll discuss in a moment.

Let's build intuition for why `3f+1` is needed.

### Why 3f+1? (Intuition, not proof)

Suppose we have `N` nodes, with up to `f` Byzantine. We want the honest majority to reach agreement despite the liars.

When you want a decision, you gather messages from the group. But:

- Up to `f` nodes might not respond (silent, crashed, partitioned).
- Up to `f` nodes might respond with lies.

So from any quorum `Q` that you wait for, you subtract:

- `f` nodes that might not be in `Q` at all (they're silent).
- `f` of the nodes *in* `Q` might be lying.

For the remaining honest-and-responsive nodes in `Q` to constitute a majority of the honest nodes overall, we need:

`|Q| - f > (N - f) / 2`

Solving: `|Q| > (N + f) / 2`, i.e., `|Q| ≥ (N + f + 1) / 2 = (N + f)/2 + 1/2` (integer math) → `Q ≥ ⌈(N+f+1)/2⌉`.

Now for two such quorums to intersect in at least `f+1` honest nodes (so that any two quorums agree on the honest majority), we need:

`2|Q| - N ≥ f + 1`

Combining gives `N ≥ 3f + 1`. Hence the magic number.

(The careful version of this argument is in the 1982 paper and in Castro and Liskov's PBFT paper. The sketch above is enough to feel *why* it's not `2f+1`.)

### Quorum sizes in BFT

With `N = 3f + 1`:

- **Quorum size** (for most BFT protocols): `2f + 1`.
- **Two quorums intersect in at least** `f + 1` **nodes**, so at least one honest node is in any intersection — enough to ensure consistency.

| `f` | `N` | Quorum |
|---|---|---|
| 1 | 4 | 3 |
| 2 | 7 | 5 |
| 3 | 10 | 7 |

The jump from `2f+1` to `3f+1` is expensive. Tolerating one failure goes from 3 nodes (crash) to 4 nodes (Byzantine). Tolerating two failures goes from 5 to 7. The proportional overhead of BFT is roughly 50% more hardware for the same fault tolerance.

## Authenticated vs. unauthenticated messages

Lamport, Shostak, and Pease distinguish two message models:

- **Unauthenticated (oral) messages.** A node receiving a relayed message has no way to verify who originally sent it. A traitor can claim "General A told me X" and there's no way to verify. This is the model in which `3f+1` is necessary.
- **Authenticated (signed) messages.** Every message is signed by its author. A relayed message can be verified: if General A signed it, no other node can forge it. In this model, **`f + 1` nodes suffice for some variants of the problem**, though the general case still depends on network assumptions. For the full distributed-consensus problem under partial synchrony, you still need `3f+1` nodes.

In practice, every modern BFT system (PBFT, HotStuff, Tendermint, the whole lineage) uses signatures. The `3f+1` bound still applies to the *node count*, because it's driven by quorum intersection arguments, not by message authenticity. But signatures change what a Byzantine node can do — it can still lie about what it received, but it can't impersonate other nodes or forge their votes.

This is a significant practical simplification. Without signatures, a Byzantine node can claim "I heard X from A" when A said nothing of the sort, and honest nodes have to untangle that. With signatures, every statement about another node's vote must be backed by that node's actual signature, which a Byzantine node cannot fabricate.

## What changes from CFT to BFT

Compare the required machinery:

| Aspect | CFT (Paxos/Raft/VR) | BFT (PBFT and descendants) |
|---|---|---|
| Node count | `2f+1` | `3f+1` |
| Quorum | `f+1` | `2f+1` |
| Message authenticity | MACs or TLS suffice | Full digital signatures needed |
| Phase count per decision | 2 (prepare, accept) | 3 (pre-prepare, prepare, commit) or more |
| Why 3 phases | N/A | To prevent equivocation — a primary sending conflicting pre-prepares to different nodes |
| Client confirmation | 1 replica's reply | `f+1` matching replies (so at least one honest) |
| Message complexity | `O(n)` per decision | `O(n^2)` classical; `O(n)` with linear BFT |

The three-phase protocol is specifically to handle a Byzantine primary. A CFT leader might crash, but it won't *lie* about what it proposed. A Byzantine primary might propose X to half the replicas and Y to the other half — equivocation. The extra phase exists to let replicas cross-check each other before committing.

## What stays the same

A surprising amount:

- **State machine replication as the goal.** Same problem, same decomposition.
- **View/ballot/term as a logical clock for leadership.** PBFT calls it a view, just like VR.
- **Leader changes as a first-class subroutine.** PBFT's view change is structurally similar to VR's, plus extra cross-checking for Byzantine primaries.
- **Commit rules driven by quorum intersection.** Everything still grounds in "any two quorums overlap in enough correct nodes."
- **Partial synchrony for liveness.** BFT also inherits the FLP escape hatch; it's safe under asynchrony, live under partial synchrony.

If you understand Paxos/Raft/VR deeply, BFT feels like "the same thing, but everyone's paranoid."

## The variants of the generals' problem

The 1982 paper actually presents several versions, with subtly different requirements:

1. **Byzantine Agreement (BA).** One distinguished *commander* starts with a value. All loyal lieutenants must agree on the same value. If the commander is loyal, the agreed value must be the commander's.
2. **Interactive Consistency (IC).** Every node starts with a value. All loyal nodes must agree on a *vector* of values, one per node, such that the entry for each loyal node is that node's input.
3. **Byzantine Generals Problem (BGP).** The general one, essentially BA without designating a commander.

For distributed-systems purposes, BA is the foundational case — "the leader has a value, everyone agrees on it (or detects the leader as faulty)." State machine replication is BA applied repeatedly, with the primary as commander.

## Where this leaves us

The classical Byzantine results are forty-plus years old. They establish:

- The node count needed (`3f+1`).
- The message authentication model options.
- The formal definition of what "agreement under lying" means.

What they don't give you is a practical algorithm you can run on modern hardware. The 1982 paper's algorithms are pedagogical — they show existence, not performance. For decades after, BFT was "theoretically solved but practically intractable": the message complexity was `O(n^{f+1})` or worse, making it useless beyond tiny `n`.

That stayed the state of the art until 1999, when Miguel Castro and Barbara Liskov published *"Practical Byzantine Fault Tolerance"*, which reduced the cost enough to actually run. PBFT was the breakthrough that made BFT useful, and it is the direct ancestor of nearly every modern BFT system.

That's the next chapter.
