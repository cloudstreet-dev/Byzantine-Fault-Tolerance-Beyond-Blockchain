# Viewstamped Replication

Barbara Liskov and Brian Oki published Viewstamped Replication (VR) in 1988, at the same PODC where Dwork, Lynch, and Stockmeyer presented their partial synchrony results. Paxos was a year away from Lamport's first draft. Viewstamped Replication was the first complete, workable, crash-fault-tolerant consensus-based replication protocol.

And then it got quietly forgotten.

It came back, three decades later, when Raft's authors cited it as a significant influence — much more so than Paxos. In 2012, Liskov and Cowling published *Viewstamped Replication Revisited* (VRR), updating the protocol for modern systems. If you squint, Raft is more a descendant of VRR than of Paxos.

This chapter is a bit shorter than Paxos and Raft, because much of VR will feel familiar. We focus on what VR teaches that the other two don't emphasize: the explicit **view change** protocol, the **primary/backup** framing, and the treatment of **client requests** as first-class concerns.

## The setup

VR has `N = 2f + 1` replicas, tolerating `f` failures. At any moment, one replica is the **primary**; the rest are **backups**. A **view** is a period during which a particular primary is in charge. Views are numbered with strictly-increasing **view numbers**. When a primary fails or is suspected of failing, the remaining replicas execute a view change protocol, electing a new primary and incrementing the view number.

Compare:

| Raft term | VR term | Paxos term |
|---|---|---|
| Term | View | Ballot |
| Leader | Primary | Distinguished proposer |
| Follower | Backup | Acceptor |
| Log | Op-log | Sequence of slot decisions |
| AppendEntries | Prepare | Phase 2 Accept |

These are different names for largely the same mechanism. But VR's choice of vocabulary — primary/backup, view, op-log — makes the protocol read less like *agreement* and more like *replication of a service*. That framing, turns out, is what real systems care about.

## Normal operation

A client sends a request to the primary. The primary:

1. Assigns the request an **op-number** (its position in the op-log).
2. Appends `(view, op-number, request)` to its log.
3. Sends `Prepare(view, op-number, request, commit-number)` to each backup.

Each backup, on receiving a `Prepare`:

1. Verifies the view is current and the op-number is next in sequence (waiting or requesting state transfer if not).
2. Appends the entry to its log.
3. Persists (with the same fsync caveats from Chapter 3).
4. Replies `PrepareOK(view, op-number, replica-id)`.

The primary commits the op once it has `PrepareOK` from `f` backups (so `f+1` total replicas, counting itself, have the op). It then:

1. Updates its commit-number.
2. Executes the op against the state machine.
3. Replies to the client.
4. Piggybacks the new commit-number on the next `Prepare` (or sends an explicit `Commit` message if there's no pipelined traffic).

Backups execute committed ops against their state machines in op-number order.

This is identical in structure to Raft's normal operation: one round-trip to a majority, leader applies, followers apply as they learn the commit.

## Client tables

VR makes explicit something Raft leaves to the application: each client has a monotonically increasing **request-number**, and the primary keeps a **client table** mapping each client to its latest request-number and the result of that request. If a client retries a request, the primary can short-circuit — look up the table, return the cached response — without redoing the work.

This matters because consensus without deduplication is not linearizable: a client that times out and retries can cause its command to execute twice. The client table is what makes VR's at-most-once semantics clean. In Raft, you implement this yourself; in VR, it's built in.

## View change

The view change is VR's most educational contribution. When backups suspect the primary has failed (by timeout), they initiate a view change:

1. Each replica increments its view number, enters `view-change` status, and sends `StartViewChange(new-view, replica-id)` to everyone.
2. On receiving `StartViewChange` from `f` other replicas (so `f+1` total in agreement), a replica sends `DoViewChange(new-view, log, old-view, op-number, commit-number, replica-id)` to the *proposed new primary* — the replica whose id is `new-view mod N`.
3. The new primary waits for `DoViewChange` messages from `f` other replicas. It now has the logs from `f+1` replicas (including itself).
4. The new primary chooses the log from whichever replica has the highest-view-number entries (ties broken by largest op-number). This is the definitive log.
5. The new primary sends `StartView(new-view, log, op-number, commit-number)` to the backups.
6. Backups replace their logs with the new primary's, enter the new view, and resume normal operation.

Three properties fall out:

- **Safety.** Any op that was committed in a prior view was, by the commit rule, replicated to at least `f+1` replicas. Any `DoViewChange`-gathering quorum of `f+1` replicas intersects that quorum in at least one replica. The new primary sees the op and includes it.
- **Liveness.** Under partial synchrony, view changes eventually succeed: messages arrive, timeouts are bounded, the new primary establishes itself.
- **View number monotonicity.** A replica in view `v` never regresses. A `Prepare` for view `v' < v` is rejected.

Raft's election + log-completeness check is a streamlined version of this: the election is the `StartViewChange` + `DoViewChange` combination, and the "at least as up-to-date" rule is how Raft picks the new leader's log without needing to gather logs from everyone explicitly.

## Recovery

When a replica crashes and comes back up, it can't just resume. Its durable state might be stale. VR specifies a **recovery protocol**:

1. The recovering replica picks a fresh nonce `x` and sends `Recovery(replica-id, x)` to all replicas.
2. Each other replica responds with `RecoveryResponse(view, x, log, op-number, commit-number)` if it is the primary, or a simpler response with just view and nonce if it is a backup.
3. The recovering replica waits for responses from `f+1` replicas (including the current primary's if available). It adopts the current view and, if the primary responded, the log.
4. It resumes as a backup.

The nonce prevents an attacker or stale state from sneaking old recovery responses in as current. This is a defense against replay; VR takes this sort of thing seriously because it was designed for research settings where replicas were actually distributed across untrusted administrative boundaries at times.

## State transfer

A replica that falls behind (via a long delay or after recovery) catches up via **state transfer**. It asks another replica for the missing log entries between its op-number and the current commit-number. The other replica ships them.

This is functionally identical to Raft's catch-up mechanism, but VR makes it explicit in the protocol. It's one more piece you don't have to invent when implementing the algorithm.

## What VR gets right that neither Paxos nor Raft emphasizes

Reading the VR papers and comparing them to Paxos and Raft, a few things stand out.

**Client semantics are in the protocol.** The client table, request-numbers, at-most-once delivery — all baked in. You don't read three VR implementations and see three different deduplication schemes; you see one.

**The primary/backup framing is the right mental model.** It matches how practitioners think about replicated services. When you say "primary/backup," engineers know roughly what you mean. When you say "proposer/acceptor" or "leader/follower," the meaning is the same but the intuition takes longer to build. VR's naming reduces the activation energy.

**View change is treated as a first-class subroutine.** The messages are named (`StartViewChange`, `DoViewChange`, `StartView`), the sequencing is explicit, and the correctness rule ("adopt the log from the replica with the highest-view latest op") is clean. Raft has this, but as part of the election plus AppendEntries pipeline; VR separates concerns.

**Recovery is specified.** The protocol says what a replica coming back from a crash does, step by step, with a nonce against replay. Raft assumes you persist `currentTerm`/`votedFor` and can resume; the rest is implicit. VR's explicitness is easier to implement correctly.

## Relationship to Raft

Ongaro's dissertation credits VR more than Paxos, and it shows. Raft's:

- Strong-leader design: inherited from VR's primary/backup.
- Log-matching invariant: VR has the equivalent via `DoViewChange` and the primary's log-merge rule.
- Explicit membership change via a reconfiguration entry in the log: VR has an equivalent.
- Catch-up via the same AppendEntries messages that carry new entries: Raft's slight innovation; VR uses separate state transfer.

If you already know Raft, VR reads like a well-documented alternate history. If you are new to both, reading VRR (the 2012 *Revisited* paper) and then Raft gives you a clearer view of the problem space than either alone.

## Why VR was "lost"

A guess, and no more: Paxos had Lamport's reputation behind it, and a paper with his name on the front. Viewstamped Replication had two MIT researchers and an unfamiliar framing (it was originally aimed at replicated services in a slightly different model — *replication through agreement on a sequence of views*). Paxos was mysterious but famous; VR was concrete but less cited. By the time the community realized that concrete beats mysterious, a decade had passed and the algorithm was "old."

The 2012 VRR paper was partly an acknowledgment that VR deserved re-introduction. It did not have Raft's cultural impact, but it should be read by anyone serious about implementing consensus. It is shorter and more complete than most of the Paxos canon.

## What you should take from VR

- There is rarely one "right" algorithm; there are multiple coherent designs that tolerate the same failures. Paxos and VR were contemporaneous, solved the same problem, and pick different emphases.
- **The client-facing parts matter.** Request numbers, client tables, at-most-once semantics — if these aren't in your algorithm, they'll be in your bug tracker.
- **View changes are not scary if you name them.** The VR view change is four messages long. Raft's election is also short. The length of these protocols is the length of a careful state diagram, not an impossibility.
- If you ever need to defend a choice of "we used Raft" to a reviewer, the honest answer is: "It's Viewstamped Replication with better docs and a better name." Nobody will be mad.

Next, we turn to the dark side: what happens when nodes don't just crash, but lie.
