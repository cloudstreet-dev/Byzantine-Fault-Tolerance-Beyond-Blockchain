# PBFT: Practical Byzantine Fault Tolerance

In 1999, Miguel Castro and Barbara Liskov published *Practical Byzantine Fault Tolerance* at OSDI. The title is a statement. The paper's central claim was that BFT had finally become cheap enough to use — that a BFT-replicated NFS was only about 3% slower than an unreplicated one. For a field that had been assuming BFT was academically interesting but operationally prohibitive, this was a sea change.

Nearly every Byzantine-fault-tolerant system you can name descends from PBFT. Tendermint is PBFT with some changes. HotStuff is PBFT refactored for linear communication. LibraBFT, DiemBFT, AptosBFT — all HotStuff descendants, so all PBFT's grandchildren. Hyperledger Fabric's BFT ordering service is a PBFT variant. If you understand PBFT, you have the backbone of modern BFT.

## The setting

- `N = 3f + 1` replicas, numbered `0..N-1`.
- Up to `f` replicas may be Byzantine.
- Replicas are linked by an asynchronous network (messages can be delayed but not forged, assuming signatures).
- Messages are authenticated — signatures or MAC-authenticators.
- At any time, one replica is the **primary**, the rest are **backups**.
- The primary for view `v` is replica `p = v mod N`.
- A **view** is a period during which a particular primary is in charge.

## Normal operation: three phases

The three-phase protocol is the heart of PBFT. Clients send requests to the primary; the primary drives them through three phases; replicas commit and reply.

```
client   primary  backup  backup  backup
   │      │        │       │       │
   │──req─▶│        │       │       │
   │      │──pre-prepare───▶│─────▶│──┐
   │      │                 │       │  │
   │      │◀────prepare─────┤       │  │  Phase 2: everybody broadcasts
   │      │◀────prepare─────┼──────┤  │           to everybody
   │      │───prepare──────▶│──────▶│  │
   │      │                 │       │  │
   │      │──────commit─────┼──────▶│
   │      │◀─────commit─────┤       │
   │      │                 │       │
   │◀──reply─────────────────       │
```

In more detail:

**Phase 1: Pre-prepare.** The primary assigns the request a sequence number `n` and sends `PRE-PREPARE(v, n, d, m)` to all backups, where `v` is the view, `n` the sequence number, `d` the digest of the request, and `m` the request itself. The primary is committing to the assignment: "this request goes in slot `n` of the log."

**Phase 2: Prepare.** Each backup, if it accepts the pre-prepare, sends `PREPARE(v, n, d, i)` to all other replicas (including the primary), where `i` is its own id. A replica **prepares** a request when it has collected `2f` matching `PREPARE` messages (from itself and `2f` others, so `2f+1` total agree on the `(v, n, d)` tuple).

Prepare is what ensures no two conflicting requests can be prepared under the same view and sequence number. It needs `2f+1` nodes to agree — a quorum of honest-plus-possibly-lying nodes that, by quorum intersection, share honest witnesses with any other quorum.

**Phase 3: Commit.** Having prepared, each replica sends `COMMIT(v, n, d, i)` to all others. A replica **commits** when it has collected `2f+1` matching `COMMIT` messages. Once committed locally, it executes the request and replies to the client.

Commit is what ensures a committed request survives *view changes*. Prepare alone would be enough for agreement within a view, but a Byzantine primary could tell different replicas that different values were prepared. The commit phase forces cross-replica confirmation that a value is not only locally prepared but visible-as-prepared to `2f+1` replicas. Any future view change will see at least `f+1` honest replicas with that committed value, and they will propagate it.

## Why three phases

Why do we need *three* phases instead of Raft's two?

Consider a Byzantine primary. In Raft, the leader is trusted: it might crash, but it won't lie. In PBFT, the primary can equivocate — send `PRE-PREPARE(v, n, d_1, m_1)` to some replicas and `PRE-PREPARE(v, n, d_2, m_2)` to others, with `d_1 ≠ d_2`. This is a Byzantine primary trying to get two different values committed for the same slot.

The **prepare** phase defeats this. A replica only prepares a request if it sees `2f+1` matching prepares — meaning that at least `f+1` honest replicas agree on the same digest. Two different digests couldn't both collect `2f+1` prepares, because the quorums overlap in at least `f+1` honest replicas, who wouldn't prepare two different digests.

So after prepare, one can say: "at most one digest could have been prepared in view `v` at slot `n`."

Why then commit? Because prepare-only is insufficient under view change. Suppose value X is prepared in view 1 — `2f+1` replicas have matching prepares. A view change happens. The new primary gathers state from `2f+1` replicas. Only `f+1` might be among those who have the prepared X; an adversary (or just bad luck) could arrange that the gathered `2f+1` includes fewer than `f+1` honest-and-with-X replicas, and X gets lost.

Commit forces a second round of confirmation: by the time a replica *commits* X, it knows that `2f+1` replicas agree X was prepared, and any future view change's gathered state must include at least `f+1` honest replicas who saw that agreement. The new primary reconstructs the committed value.

In short: prepare ensures agreement within a view; commit ensures agreement across views.

## Client semantics

A client sends its request to the primary (or all replicas, if the primary is suspected faulty). The client waits for `f+1` matching replies, so that at least one honest replica vouches for the result. `f+1` is the minimum such that, even if `f` are Byzantine and forge replies, at least one honest-and-correct reply is in the set.

Clients maintain a *timestamp* (monotonically increasing per client) to deduplicate requests. Replicas keep a per-client table, similar to VR's client table, mapping client id to the latest timestamp executed.

## View change

When backups suspect the primary is faulty (typically via a timeout on a request they forwarded), they initiate a view change.

1. Each backup enters `view-change` status and sends `VIEW-CHANGE(v+1, n, C, P, i)` to all replicas, where:
    - `v+1` — the new view.
    - `n` — the highest sequence number the replica has executed (stable checkpoint).
    - `C` — the set of checkpoint certificates (proofs that the state up to `n` is agreed).
    - `P` — for each request prepared since the last stable checkpoint, a certificate of that preparation (a set of `2f` matching `PREPARE`s plus the `PRE-PREPARE`).
2. The new primary for `v+1` (which is `(v+1) mod N`) collects `VIEW-CHANGE` messages from `2f` replicas (plus its own, so `2f+1` total).
3. The new primary constructs a `NEW-VIEW(v+1, V, O)` message, where:
    - `V` is the set of valid `VIEW-CHANGE` messages it received.
    - `O` is a set of `PRE-PREPARE` messages for sequence numbers to be re-proposed in the new view. For each sequence number `n` in the relevant range, if any of the received `VIEW-CHANGE`s showed a prepare certificate for `n` with digest `d`, the new primary issues `PRE-PREPARE(v+1, n, d)`. For sequence numbers with no prepare certificate, it issues a null pre-prepare (a placeholder no-op).
4. The new primary broadcasts `NEW-VIEW` along with all the `PRE-PREPARE`s.
5. Backups validate the `NEW-VIEW`: they check that it is consistent with the `VIEW-CHANGE` messages, that the `PRE-PREPARE`s correctly reflect prepared values from prior views, and that all relevant sequence numbers are handled. If valid, they enter `v+1` and proceed with normal operation.

The complexity here is real. View change is the hardest part of PBFT to implement correctly; it has an enormous set of invariants to preserve. The original paper and Castro's thesis spend many pages on it. It is the single place where a Byzantine primary can cause the most havoc, and it's the place where the protocol's correctness is tightest.

### Why view change is so hairy

A view change must:

- Ensure every committed request from prior views is preserved.
- Rebuild the log for sequence numbers that were prepared but not committed.
- Not introduce any new conflicts with past prepares.
- Do all this despite the fact that the new primary (and up to `f-1` other replicas) may be Byzantine.

Each received `VIEW-CHANGE` from a backup carries cryptographic proofs — signed `PRE-PREPARE`s and `PREPARE`s — that the new primary must validate. Replicas validate the `NEW-VIEW` by re-performing the same logic. This is a lot of crypto-checking and state-stitching.

A later revision of the protocol (Castro's dissertation) simplifies and tightens this. Modern descendants have found further simplifications — HotStuff's chained-commit structure, in particular, folds view changes into regular message flow without needing a separate hairy subroutine. We'll meet that in the next chapter.

## Checkpoints and garbage collection

The log of requests grows forever without garbage collection. PBFT uses **checkpoints**: every `k` sequence numbers (typically every 100 or 128), each replica computes a digest of its state and sends `CHECKPOINT(n, d, i)`. A replica considers a checkpoint **stable** when it has `2f+1` matching checkpoint messages. Everything before a stable checkpoint can be discarded — including prepared-but-not-committed entries (which, at a stable checkpoint, either committed or will never commit).

Stable checkpoints are important not just for storage but for bounding view change cost: the `VIEW-CHANGE` message only has to carry prepare certificates since the latest stable checkpoint, not the whole log.

## The O(n²) problem

PBFT's normal-case message complexity per decision:

- Pre-prepare: primary → `N-1` backups. `N-1` messages.
- Prepare: each of `N` replicas → `N-1` others. `N(N-1)` messages.
- Commit: each of `N` replicas → `N-1` others. `N(N-1)` messages.

Total: `O(N^2)` per decision. At `N = 4`, that's about 16 messages per commit. At `N = 100`, 10,000 messages. At `N = 1000`, a million.

This quadratic behavior is tolerable at small scale — BFT systems historically ran with 4, 7, or at most a few dozen nodes. It becomes prohibitive at blockchain scales, where validator sets of 100+ are common. This is exactly what HotStuff addresses.

The quadratic cost comes from the all-to-all broadcasts in prepare and commit. If you could somehow collapse those into linear communication while preserving the safety guarantees, you'd have PBFT with much better scaling. That's what HotStuff does, by using threshold signatures and a leader that aggregates votes.

## Optimizations in the original paper

Castro and Liskov did not stop at the pedagogical version. The OSDI 1999 paper and Castro's PhD thesis include optimizations that make real PBFT deployments workable:

- **MAC-based authentication for normal case.** Digital signatures are expensive; MAC vectors are cheap. PBFT uses MACs for pre-prepare/prepare/commit in the normal case and reserves signatures for view change messages (where non-repudiation matters most).
- **Batching.** The primary batches multiple client requests into a single pre-prepare.
- **Read-only optimization.** Read-only operations can skip the three-phase protocol — the client sends the read to all replicas, waits for `2f+1` matching replies, and is done. This is safe because reads don't modify state.
- **Tentative execution.** Replicas execute requests after the prepare phase (tentatively) and send replies to the client. A client waits for `2f+1` matching tentative replies; if it gets them, it has a strong enough certificate to proceed without waiting for commit. If not, it waits for `f+1` matching committed replies.

These optimizations cut the latency of normal operations substantially — from 3 phases to "1 phase for reads, 2 phases for common-case writes."

## Engineering lessons from PBFT

- **Byzantine safety is composable.** PBFT's three-phase protocol, once you understand it, is a reusable building block. You can run it for each log slot (linearly) or for each round of a pipeline (chained).
- **View change is where the bodies are buried.** Most PBFT implementation bugs have been found in view change. Anyone implementing BFT should write the most careful tests here.
- **Crypto costs matter.** The MAC vs. signature choice had large performance consequences. Modern threshold signature schemes (BLS, etc.) have changed this balance again.
- **BFT is not binary.** "BFT" can mean a 4-node system handling 1 failure or a 100-node system handling 33 failures. The cost structure is very different.

## What to take into HotStuff

PBFT's normal-case sequence (pre-prepare, prepare, commit) maps directly onto HotStuff's chained phases (prepare, pre-commit, commit, decide, but pipelined). The reason HotStuff can claim "linear communication" while PBFT cannot is that HotStuff aggregates votes at the leader using threshold signatures, rather than having all replicas gossip with all others.

If PBFT is the O(n²) landmark, HotStuff is the O(n) response. We'll see how next.
