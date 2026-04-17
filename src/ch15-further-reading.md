# Further Reading

Every algorithm, theorem, and system in this book came out of someone else's work. Here are the places to go when you want the source material, the fuller treatment, or the currents of ongoing research.

## Foundational papers

### Consensus and state machine replication

- **Leslie Lamport, *"Time, Clocks, and the Ordering of Events in a Distributed System,"* Communications of the ACM, 1978.** The paper that put logical clocks on the map and framed SMR. Also the one that introduced much of the vocabulary we still use.
- **Leslie Lamport, *"The Part-Time Parliament,"* ACM TOCS, 1998.** The original Paxos paper. Written as a parable, delightful once you have the context, but famously hard to learn from cold.
- **Leslie Lamport, *"Paxos Made Simple,"* SIGACT News, 2001.** The gentler presentation. Read this before *The Part-Time Parliament*.
- **Leslie Lamport, *"Fast Paxos,"* Distributed Computing, 2006.** The optimization. Usually more interesting to read about than to implement.
- **Brian Oki and Barbara Liskov, *"Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems,"* PODC 1988.** VR, contemporaneous with Paxos.
- **Barbara Liskov and James Cowling, *"Viewstamped Replication Revisited,"* MIT-CSAIL-TR-2012-021, 2012.** VRR. If you read one VR paper, read this one.
- **Diego Ongaro and John Ousterhout, *"In Search of an Understandable Consensus Algorithm,"* USENIX ATC 2014.** Raft. The readable paper that changed the field.
- **Diego Ongaro, *"Consensus: Bridging Theory and Practice,"* PhD dissertation, Stanford, 2014.** Raft in more detail, with the full formal treatment and implementation lessons.
- **Fred Schneider, *"Implementing Fault-Tolerant Services Using the State Machine Approach: A Tutorial,"* ACM Computing Surveys, 1990.** The SMR survey that taught the community how to think about the pattern.

### Impossibility results

- **Michael Fischer, Nancy Lynch, and Michael Paterson, *"Impossibility of Distributed Consensus with One Faulty Process,"* JACM, 1985.** FLP. The paper itself is short and remarkably readable.
- **Eric Brewer, *"Towards Robust Distributed Systems,"* PODC 2000 keynote.** The slides that launched CAP.
- **Seth Gilbert and Nancy Lynch, *"Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services,"* SIGACT News, 2002.** The formal CAP proof.
- **Cynthia Dwork, Nancy Lynch, and Larry Stockmeyer, *"Consensus in the Presence of Partial Synchrony,"* JACM, 1988.** Partial synchrony as a model. The backbone of "safe in asynchrony, live in partial synchrony."

### Byzantine fault tolerance

- **Leslie Lamport, Robert Shostak, and Marshall Pease, *"The Byzantine Generals Problem,"* ACM TOPLAS, 1982.** The original.
- **Miguel Castro and Barbara Liskov, *"Practical Byzantine Fault Tolerance,"* OSDI 1999.** PBFT. Short, dense, worth multiple reads.
- **Miguel Castro, *"Practical Byzantine Fault Tolerance,"* PhD dissertation, MIT, 2001.** The full treatment, with all the optimizations and engineering details the conference paper elided.
- **Maofan Yin et al., *"HotStuff: BFT Consensus in the Lens of Blockchain,"* PODC 2019.** HotStuff.
- **Shehar Bano et al., *"SoK: Consensus in the Age of Blockchains,"* AFT 2019.** A systematic survey of consensus through the blockchain-era lens.

### Randomized consensus

- **Michael Ben-Or, *"Another Advantage of Free Choice: Completely Asynchronous Agreement Protocols,"* PODC 1983.** Short, beautiful.
- **Michael Rabin, *"Randomized Byzantine Generals,"* FOCS 1983.** The shared-coin approach.
- **Christian Cachin, Klaus Kursawe, and Victor Shoup, *"Random Oracles in Constantinople: Practical Asynchronous Byzantine Agreement Using Cryptography,"* PODC 2000.** Threshold cryptography for asynchronous BFT.
- **Andrew Miller et al., *"The Honey Badger of BFT Protocols,"* CCS 2016.** Practical asynchronous BFT.

### Production systems papers

- **James Corbett et al., *"Spanner: Google's Globally-Distributed Database,"* OSDI 2012.** The TrueTime paper.
- **Patrick Hunt et al., *"ZooKeeper: Wait-free Coordination for Internet-scale Systems,"* USENIX ATC 2010.** ZooKeeper's design and Zab.
- **Giuseppe DeCandia et al., *"Dynamo: Amazon's Highly Available Key-value Store,"* SOSP 2007.** The Dynamo paper. The alternate path.
- **Jason Baker et al., *"Megastore: Providing Scalable, Highly Available Storage for Interactive Services,"* CIDR 2011.** Google's pre-Spanner Paxos-backed store.

## Textbooks

- **Christian Cachin, Rachid Guerraoui, and Luís Rodrigues, *Introduction to Reliable and Secure Distributed Programming*, 2nd ed., 2011.** The textbook if you want formal treatment. Algorithms specified precisely; correctness proved. Chapter 5 on randomized consensus is especially good.
- **Hagit Attiya and Jennifer Welch, *Distributed Computing: Fundamentals, Simulations, and Advanced Topics*, 2nd ed., 2004.** A classical theory-of-distributed-computing textbook. Good for the formal foundations.
- **Martin Kleppmann, *Designing Data-Intensive Applications*, O'Reilly, 2017.** The book to hand to a software engineer who wants one book on this space. Broader than consensus — covers databases, stream processing, batch — but the chapters on consistency, replication, and consensus are the best short treatment in the field.
- **Maarten van Steen and Andrew Tanenbaum, *Distributed Systems*, 4th ed., 2023.** A textbook in the classical mold; covers consensus alongside everything else.
- **Peter Alvaro (editor), various lecture notes and reading lists** from the distributed systems courses he has taught, which rotate through current material.

## The Jepsen reports

Kyle Kingsbury's Jepsen analyses ([jepsen.io/analyses](https://jepsen.io/analyses)) are essential reading. If you implement or operate distributed systems, read ten of these cover to cover. You will come away with a concrete sense of how claims fail under real pressure.

Recommended starting points:

- The early MongoDB reports (consistency evolution over the years).
- The etcd and Consul reports (what CFT correctness looks like when verified).
- The FoundationDB "passed" report (what thorough testing *from the authors' side* enables).
- The Redis reports (what happens when replication is bolted on later).
- The CockroachDB, YugabyteDB, TiDB, and FaunaDB reports (modern consensus databases in the crucible).

Each report follows a similar structure: "here is the system, here is the failure model I tested it under, here is what went wrong, here is the vendor's response." The cumulative effect of reading many of them is a kind of vaccination against overconfidence.

## Formal methods

If you want to convince yourself (or others) of a protocol's correctness:

- **Leslie Lamport's TLA+ suite** — [lamport.azurewebsites.net/tla](https://lamport.azurewebsites.net/tla/tla.html). Specifications in TLA+, model checking with TLC. Nontrivial learning curve, enormous payoff.
- **The Raft TLA+ specification** — by Diego Ongaro, available with the Raft paper's artifacts. A small, readable reference.
- **MongoDB's formal methods work** — public write-ups of using TLA+ and p-based methods to find bugs before shipping. Useful case studies of formal methods in industrial practice.

## Courses and lectures

- **MIT 6.824, Distributed Systems** — [pdos.csail.mit.edu/6.824](https://pdos.csail.mit.edu/6.824/). Lecture videos and labs are on the open internet. The labs walk you through implementing Raft step by step. This is how most of the industry's younger generation actually learned Raft.
- **Martin Kleppmann's distributed systems lectures (Cambridge)** — on YouTube and [github.com/mkleppmann/distsys-class](https://github.com/ept/dist-sys). Excellent, Socratic.

## The blogs and writeups

A rotating cast, but a few durable sources:

- **Murat Demirbas's blog, *Metadata*.** Paper summaries and commentary from a working distributed systems researcher.
- **Aphyr's blog** (Kingsbury's Jepsen home).
- **High Scalability archives** — historical glimpses of production architectures.
- **Google's and Meta's engineering blogs** — occasional deep dives on production systems.

## Companion volume

*How Blockchains Actually Work (Without the Hype)* — for the other half of the consensus story, where membership is open and identities are pseudonymous. It covers Nakamoto consensus, proof-of-work, proof-of-stake, and the economic-security tradeoffs that define public blockchains.

## Wrap

The literature on consensus is enormous and still growing. If you read only three papers: Lamport's *Paxos Made Simple*, Ongaro's *Raft*, and Castro and Liskov's *PBFT*. If you read only one book: Cachin, Guerraoui, and Rodrigues (for formal) or Kleppmann (for practical). If you read only one series: Jepsen.

Everything else is optional but rewarding. The field is one of the most intellectually honest corners of computer science — every bad paper gets attacked, every good paper gets extended, every algorithm gets tested. Join the conversation by reading widely and implementing carefully.

That's the book. Thank you for reading.
