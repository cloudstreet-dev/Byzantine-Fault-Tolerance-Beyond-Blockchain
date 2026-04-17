# Raft: Paxos for Humans

In 2014, Diego Ongaro and John Ousterhout published *In Search of an Understandable Consensus Algorithm* — better known by its eventual name: **Raft**. The opening paragraph does something rare for a distributed systems paper: it names its goal as *understandability*.

That framing was radical. Other consensus algorithms had been optimized for provable correctness, theoretical elegance, or asymptotic performance. Raft optimized for *humans reading and implementing it*. The authors argued, convincingly, that an algorithm no one can implement correctly is an algorithm no one is actually using.

They were right. Within a few years of the paper, Raft was everywhere — etcd, Consul, CockroachDB, TiKV, RethinkDB, the Kafka controller (eventually), and dozens of smaller projects. Meanwhile Paxos, despite being older and better-studied, continued to live mainly in Google's internal systems and a handful of determined academic implementations.

This chapter walks through Raft the way it should be walked through: not as "Paxos, but easier," but as a coherent design with its own choices.

## The three subproblems

Raft decomposes consensus into three subproblems that can be understood (mostly) independently:

1. **Leader election.** Exactly one server is in charge at a time.
2. **Log replication.** The leader replicates its log to followers.
3. **Safety.** A committed entry never gets uncommitted, even across leader changes.

The genius of Raft is that each of these is a clean, intuitive protocol in isolation, and the interaction between them — while subtle — is confined to a few specific rules.

Every Raft server is in one of three states at any time:

- **Follower** — passive, responds to leaders and candidates.
- **Candidate** — actively seeking election.
- **Leader** — handling client requests, replicating the log.

```
  ┌─────────┐ timeout  ┌───────────┐ majority  ┌────────┐
  │Follower │─────────▶│ Candidate │──────────▶│ Leader │
  └─────────┘          └───────────┘           └────────┘
       ▲                    │                      │
       │           discover │                      │
       └────────────────────┴──────────────────────┘
                  higher term / new leader
```

## Terms: Raft's logical clock

Raft numbers time with **terms**. Each term begins with an election. If the election succeeds, the winner leads for the rest of that term. If the election fails (split vote), the term ends with no leader and a new term starts.

Every RPC carries a term number. The rule is universal: **if a server receives any message with a term higher than its own, it updates its term and becomes a follower.** This single rule does an enormous amount of work. It ensures that stale leaders step down as soon as they talk to any node that knows about the new term. It prevents a leader coming back from a long pause from issuing any commands, because its next communication will get a higher-term response.

Terms map directly onto Paxos's ballot numbers and VR's views. Same idea, clearer name.

## Leader election

Followers start out receiving heartbeats from the current leader. If a follower goes some election timeout (typically 150–300ms, randomized) without hearing from a leader, it becomes a candidate:

```
1.  currentTerm ← currentTerm + 1
2.  state ← candidate
3.  vote for self, count = 1
4.  reset election timer
5.  send RequestVote RPCs to all other servers
```

Other servers vote according to two rules:

1. They haven't already voted for someone else in this term.
2. The candidate's log is at least as up-to-date as theirs.

The "up-to-date" check compares last log entries: higher term wins; same term, longer log wins. This is Raft's key safety rule for elections — it ensures a leader always has all committed entries. We'll come back to why this works.

If the candidate gets a majority of votes, it becomes leader. If it receives an AppendEntries from a server with a higher or equal term, it steps down. If the election timer runs out (split vote), it starts a new term and tries again.

### Random timeouts: the anti-livelock trick

Raft uses randomized election timeouts (e.g., uniform over [150ms, 300ms]) to break symmetry. Without randomization, if all followers notice the leader is gone at the same moment, they'd all start campaigns simultaneously, split the vote, fail, and retry in lockstep — livelock.

Randomization makes it likely that one follower times out noticeably earlier than the others, becomes a candidate, and wins before the others start. Simple; effective.

## Log replication

Once elected, the leader receives client commands, appends them to its log, and replicates them to followers via `AppendEntries` RPCs.

Each log entry contains:

- `term` — the term when the entry was created
- `index` — position in the log, 1-based
- `command` — the operation to apply to the state machine

An entry is **committed** once the leader has replicated it to a majority. Once committed, the leader can safely apply the command to its state machine and reply to the client. Committed entries are eventually replicated to all followers and applied in order.

The AppendEntries RPC is a clever bit of engineering:

```
AppendEntries(leaderTerm, leaderId, prevLogIndex, prevLogTerm,
              entries[], leaderCommit)
```

- It carries heartbeat (empty `entries[]`) as well as new entries.
- It tells the follower what should be at the position just *before* the new entries (`prevLogIndex`, `prevLogTerm`). The follower refuses the RPC if its log doesn't have a matching entry there.
- It carries `leaderCommit`, telling the follower how far the leader has committed, so the follower knows how far to apply.

The `prevLogIndex`/`prevLogTerm` check gives Raft the **log matching property**: if two logs share an entry at the same index with the same term, they are identical for all prior entries. The leader and follower converge by walking backward until they find a match and then replaying forward.

```
Log matching, visually:

  leader    ...│ t1,c │ t1,c │ t2,c │ t3,c │ t3,c │ <-- leader
  follower  ...│ t1,c │ t1,c │ t2,c │ t2,x │       <-- divergence at index 4

  AppendEntries with prevLog=(index=3, term=t2): OK, matches.
  AppendEntries with prevLog=(index=4, term=t3): follower's entry is
      term t2, mismatch. Reject. Leader decrements nextIndex and retries.
```

The leader maintains a `nextIndex` for each follower — where to send next. On a rejection, decrement and retry; on success, advance.

## Safety: the subtle rules

The hard part of Raft isn't the common case; it's ensuring safety across leader changes. Three rules do the work:

**Rule 1 (Election Restriction).** A candidate only wins if its log is at least as up-to-date as the majority it requests votes from.

This ensures the new leader has all committed entries from previous terms. The proof is quorum-intersection: a committed entry was stored on a majority; any majority that votes for a new leader intersects the committing majority; by the "at least as up-to-date" rule, the new leader's log includes that entry.

**Rule 2 (Leader Append-Only).** Leaders never overwrite or delete entries in their own log; they only append.

This means a leader's log is a monotonically-growing prefix.

**Rule 3 (Don't commit entries from previous terms via counting replicas).** This is the subtle one. A leader from term `T` cannot simply replicate an entry from term `T' < T` to a majority and declare it committed. Why?

Consider: a leader in term 2 replicates entry X to a majority, crashes before committing. A new leader is elected in term 3. It finds X in its log. If it just counts replicas of X and commits once a majority has it, there's a scenario where a subsequent leader (in term 4) with a longer log but without X could overwrite X — violating safety.

The fix: a leader only commits an entry by counting replicas *for an entry from its current term.* If there are older-term entries to "rescue," the leader commits them implicitly by committing a new entry from its current term (the commit of a later entry commits everything before it).

In practice, this manifests as: a new leader immediately issues a **no-op** entry in its own term. Once that no-op is committed, everything before it is committed transitively.

The Raft paper spends pages on this, with a detailed scenario (Figure 8 of the paper). It's the single most important rule that beginners miss. Without it, Raft is subtly broken.

## Membership changes

Changing the cluster configuration (adding or removing servers) is dangerous if done naively. The old and new configurations might have overlapping but non-intersecting majorities — two different leaders could coexist, one winning the old majority, one winning the new.

Raft's approach is **joint consensus**: temporarily require majorities in both the old and new configurations for any decision. The transition proceeds in two log entries:

1. `C_old,new` — a joint configuration containing both member sets. Decisions need a majority of `old` AND a majority of `new`.
2. Once `C_old,new` is committed, append `C_new` — the new configuration alone.

During the joint period, no single configuration can decide on its own, so there can be at most one leader. Once `C_new` commits, the old configuration is abandoned.

Newer Raft implementations often use **single-server changes** (add or remove one server at a time) for simplicity. The paper proves this is safe *if* you add/remove one server at a time and the quorums of old and new differ by exactly one member, always overlapping. Most production Raft libraries use this simpler form.

## Log compaction

Logs grow; storage is finite. Raft uses **snapshots**: periodically, the state machine is serialized; everything in the log before the snapshot index can be discarded; the snapshot and a small bit of metadata (lastIncludedIndex, lastIncludedTerm) take their place.

A follower that is too far behind to catch up via the log receives an `InstallSnapshot` RPC. It installs the snapshot, discards its own log, and starts from there.

Snapshotting is the messiest practical aspect of Raft. You need:

- Consistent snapshots (usually by application-level serialization while buffering new commands).
- Atomic replacement of the snapshot file.
- Correct handling of log truncation across all servers.
- Recovery from mid-snapshot crashes.

There is nothing deep here, but there's a lot of code. Raft implementations are usually 1500-3000 lines; a chunk of that is snapshotting.

## Client interaction

Three subtleties:

1. **Retries and duplicate commands.** If a client's RPC to the leader times out, the client may retry. The command might have already been committed. Raft supports linearizability by having clients assign a unique ID per command, and the leader deduplicates based on ID. This is usually handled in the state machine, not Raft itself.
2. **Read-only queries.** Reads that need linearizability can't just hit any follower — followers might be stale. Three common approaches:
    - **Forward to leader.** Every read goes through the leader, which confirms it is still the leader (by exchanging heartbeats with a majority) and then serves the read.
    - **Leader leases.** The leader holds a time-bounded lease; as long as the lease is valid, it can serve reads without a round-trip. Requires synchronized-ish clocks, which introduces its own risks.
    - **Follower reads with read index.** A follower asks the leader for a read index, waits until its log has applied up to that index, then reads. Cheaper for the leader, slower for the follower.
3. **Leader stickiness.** Clients usually cache "the last leader I talked to" and retry there first. If that server is no longer leader, it returns a hint; clients re-resolve.

## Why Raft succeeded where pedagogical Paxos failed

A few reasons, empirical and structural:

- **Named subproblems.** Leader election, log replication, safety. Engineers can learn one at a time.
- **A single canonical algorithm, not a family.** Multi-Paxos has dialects; Raft has a reference implementation and most libraries match it closely.
- **Strong leader.** The leader is authoritative for its term; followers defer to it. This is less symmetric than Paxos but much easier to reason about.
- **Concrete RPC interfaces.** The paper gives you RequestVote and AppendEntries with field names. Paxos gives you phases. One you can implement directly; the other you have to translate.
- **The paper itself is good.** It's written like a textbook chapter, not a theorem-proof sequence. Diagrams, pseudocode with line numbers, worked examples.

The downside is that Raft's "strong leader" design can be less efficient than leaderless Paxos variants (e.g., EPaxos) under certain workloads. But the clarity-to-performance tradeoff favored clarity, and clarity won the market.

## The subtle things beginners miss

Every Raft implementation review catches the same handful of bugs. Here are the ones you should look for in your own:

1. **Committing previous-term entries by counting.** The commit rule is "current term, majority replicated." Entries from earlier terms piggyback via a current-term entry. Get this wrong and you lose writes.
2. **Voting without persisting.** `currentTerm` and `votedFor` must be persisted to disk *before* responding to a RequestVote. Otherwise a crashed-and-recovered server can vote twice in the same term.
3. **Log index off-by-ones.** Raft's log is 1-indexed. Snapshot metadata refers to the *last included* index. Every careful implementation has a set of invariants checked at each operation; every hasty implementation has a bug here.
4. **Election timer reset conditions.** The timer resets on: receiving a valid AppendEntries from the current leader, granting a vote. It does NOT reset on: receiving an AppendEntries with a stale term, receiving a RequestVote you deny. Get this wrong and you get leader flapping.
5. **Uncommitted-to-committed transitions not atomic w.r.t. state machine.** The state machine only applies committed entries, in order, one at a time. A restart must replay from the last applied index. Miss this and you apply commands twice.

Most production implementations have tests for each of these (good ones have Jepsen verdicts). Fewer students' implementations do. This is the gap between "I implemented Raft" and "I shipped Raft."

## Raft by the numbers

For a 5-node Raft cluster:

- Quorum size: 3
- Maximum failures tolerated (crash-recovery): 2
- Steady-state round trips per write: 1 (leader → followers → leader)
- Messages per write (best case): `2 * (N - 1)` = 8 at N=5
- Disk writes per committed entry: 1 on leader, 1 on each of the replicas (before ack)

You'll find that most Raft systems quote something like "~1ms writes in-cluster, ~5ms cross-AZ, ~50-100ms cross-region" as order-of-magnitude figures. These depend heavily on batching, hardware, and distance; treat them as ranges, not numbers to quote back.

## What Raft teaches

- Clarity is not a nice-to-have; it is a property of the design that determines whether an algorithm will be implemented correctly by thousands of engineers who aren't its authors.
- Strong leadership simplifies the common case. Pay the cost at leader change, earn it back every second of steady state.
- Safety reasoning should be reducible to a small number of invariants. Raft has three; Paxos has two; PBFT has roughly four. If you can't list your algorithm's safety invariants in a short paragraph, you probably don't understand it yet.
- Pseudocode with line numbers beats prose descriptions. Always.

Next: Viewstamped Replication. The algorithm that was there first, and that teaches you things neither Paxos nor Raft does.
