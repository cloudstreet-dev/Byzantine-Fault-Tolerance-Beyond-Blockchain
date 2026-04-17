# Introduction: The Book Blockchain Wasn't

If you read the news between 2016 and 2022, you could be forgiven for thinking that blockchain had invented distributed consensus. Every think piece about supply chains, elections, medical records, and loyalty points reached for the same vocabulary — miners, validators, gas, proof-of-work, proof-of-stake — as if these were the primitives of agreement itself.

They are not.

Consensus — getting a group of computers to agree on a single answer despite failures, delays, and concurrency — is a solved problem in computer science, and it was solved decades before Bitcoin. The catch is that the classical solutions assume something blockchain refuses to: that you know who is in the room. When you control membership, the problem gets dramatically easier. You need fewer messages. You don't need economic incentives. You don't need to boil oceans. You get stronger guarantees, faster finality, and cheaper operation.

This is the book about those solutions. The ones running inside etcd, which in turn runs Kubernetes, which in turn runs most of the world's production software. The ones inside Consul, ZooKeeper, the Kafka controller (now KRaft), Spanner, CockroachDB, FoundationDB, and the configuration plane of nearly every cloud. The ones in permissioned financial networks that happen to call themselves blockchains because the word polled well with executives. And the research lineage behind all of them: Paxos, Raft, Viewstamped Replication, PBFT, HotStuff, and their relatives.

## Who this book is for

Working engineers. You have used a distributed system. You have read a post-mortem. You know what a split-brain is, have strong opinions about retries, and are tired of being told that every problem is a blockchain problem. You want to understand, concretely, what's happening when your key-value store "achieves consensus" — enough that you can read an architecture decision record or a Jepsen report and evaluate it on its merits.

The prerequisites are modest. You should be comfortable with the idea of a state machine, know that networks can drop messages and reorder them, have at least heard the phrase *CAP theorem*, and be able to read a code snippet in a C-family language. You do not need to know any specific algorithm. You do not need formal-methods training. We translate the math into pictures and invariants a practitioner can hold in their head.

## What this book is not

It is not a blockchain book. There is a companion volume for that: *How Blockchains Actually Work (Without the Hype)*. Here, we mention blockchain only when it earns the mention — in Chapter 12, where permissioned chains overlap with classical BFT, and occasionally as a contrast class.

It is not a proofs book. Correctness arguments are given in the form an engineer can check by hand ("if two quorums overlap in at least one node, and that node refuses to accept two conflicting proposals, then..."), not the form a theorem prover can check. Pointers to the formal treatments live in Chapter 15.

It is not a library tour. We will use etcd, ZooKeeper, and their peers as case studies, but the goal is that when a new consensus-based system lands on your desk, you recognize the shape.

## How the book is built

The story is a chain of problems and responses.

1. First we establish the problem: **state machine replication** — the canonical use case that every consensus algorithm is trying to solve.
2. Then we meet the walls: **FLP** (you cannot guarantee consensus in a purely asynchronous system with even one crash) and **CAP** (you cannot have consistency, availability, and partition tolerance simultaneously). These are the constraints every practical algorithm has to route around.
3. Then we get precise about what *failure* means — **crash, omission, Byzantine** — because algorithms are cheap or expensive depending on which of these you sign up for.
4. With those in hand, we take the tour. **Paxos** (the one everyone names), **Raft** (the one everyone uses), **Viewstamped Replication** (the one that predates them both and deserves the credit), **PBFT** (when you can't trust your peers), **HotStuff** (when you have a lot of peers), and **randomized consensus** (when you want to sidestep FLP with a coin).
5. Then we walk the production floor — etcd, Consul, ZooKeeper, Kafka, Spanner, CockroachDB — and ask what these algorithms actually look like in systems people run for a living.
6. We take a fair look at permissioned blockchains.
7. We discuss what goes wrong: liveness, safety, split-brain, and the incidents that teach the theory better than the theory does.
8. We give you a decision procedure for the next time someone asks you, *"should we use Raft?"*
9. We point you at the papers and the textbooks for everything we had to cut.

## A note on stance

Every algorithm in this book exists because the previous one had a problem. Paxos exists because Lamport found a way to reach consensus despite FLP. Raft exists because Paxos-the-paper was incomprehensible enough to hold the field back for a decade. PBFT exists because crash-fault tolerance is not enough when your peers might lie. HotStuff exists because PBFT's communication costs got painful at scale.

We try to tell this story the way you'd tell it to a colleague over coffee: respectfully of the reader, unimpressed by jargon, willing to say *"the part of the original paper that confused everyone was..."*, and honest about tradeoffs. "Raft is easier to implement than Paxos" is a genuine engineering win. "PBFT needs `3f+1` nodes to tolerate `f` failures" is a real cost. Both things are true and both are interesting.

There are no revolutionary, game-changing, or next-generation algorithms here. Just the slow, patient accumulation of good ideas that keep the lights on.

Let's go.
