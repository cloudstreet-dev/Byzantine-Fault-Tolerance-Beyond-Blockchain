# Byzantine Fault Tolerance Beyond Blockchain

> Paxos, Raft, PBFT, Viewstamped Replication, HotStuff — the consensus algorithms that aren't blockchain. Real tradeoffs, real math, no hype.

**Read the book:** https://cloudstreet-dev.github.io/Byzantine-Fault-Tolerance-Beyond-Blockchain/

## Why this book

Blockchain isn't the only way to reach consensus — it's a specific design point for a specific problem (open membership, untrusted identities, public networks). When you control who's in the room, you have better options. This book is a patient tour of the consensus algorithms that actually run production databases, distributed systems, and permissioned networks.

It is a companion to [*How Blockchains Actually Work (Without the Hype)*](https://github.com/cloudstreet-dev/How-Blockchains-Actually-Work) but stands alone. A reader who knows what a hash function is and has heard the phrase "CAP theorem" can start here.

## Build locally

```sh
cargo install mdbook
mdbook serve --open
```

## License

[CC0 1.0 Universal](./LICENSE). Public domain dedication. Use it, remix it, translate it, teach from it — no permission required.
