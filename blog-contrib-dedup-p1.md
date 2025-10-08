WARNING:  This is my WIP notes and it may contain mistakes.

TODO: Change "we" to the name of the system in the blog after checking if this text is acceptable. Round the results like 28.8 GB after updating them (migrate a newer mainnet snapshot before the blog and run extra tests again while newer stats are being extracted).

## How We Cut 210M Cryptographic Hashes With Account Key Deduplication

By optimizing how we store account keys, we reduced execution state size by 28.8 GB (over 7 percent of the entire execution state). Perhaps more importantly, we did this by reducing the total number of payloads, and we also reduced the payload count's growth rate.

Reducing payload count matters because each payload can add different overhead (cpu, ram, disk, network) to different types of servers running databases, caches, execution state, indexers, etc. that handle payloads in various ways.

As one example, MTrie (the execution state), uses 2-3 trie vertices (192-288 bytes) for each payload, and that overhead is larger than the 72-78 byte payload sizes of account keys.  We eliminated over 210 million Mtrie vertices containing cryptographic hashes we don't need anymore, by reducing the number of payloads by 86.1 million.

To put what we achieved into perspective, we found 77.6 million payloads with duplicate keys, but our improved data format reduces the number of payloads by 86.1 million without using compression (not a typo)!

These incremental improvements have a cumulative impact on operational costs, uptime, and performance.  Although payload count increases daily, Execution Nodes (EN), etc. can handle more payloads now compared to 9 months ago, without adding extra RAM because there are fewer payloads now.

Reducing payload counts and their overhead helps reduce hardware costs and energy consumption on various servers that handle payloads.  By improving resource utilization, we gain headroom to handle increased spikes in demand, reduce downtime, and reduce costs.  Memory use reduction on AN, EN, etc. are hard to measure since we are deploying unrelated projects at the same time, but we know we are loading 28.8 GB less execution state into RAM on EN, and memory usage reduction on EN in a prior migration project was reported to be a multiple of the state size reduction.

At the same time, the change does not take anything away from end users. [...]

### MY NOTES

From the "Results" section of the README (also DRAFT) at:
- https://github.com/fxamacker/draft-notes-about-deduplicating-public-keys

NOTE: The new data format is designed to reduce payload count by more than the total number of duplicate keys.

There are ~77.6 million duplicate payloads and migration reduces payload count by 86.1 million payloads (not a typo).

Most accounts on mainnet only have 1 account key but they still use fewer bytes in the new data format.

|                  | Reduction | Notes |
| ---------------------- |--------| --- |
| Public Keys | 77.6 million (53.1%) | better than our goal of 46-48% 🎉 |
| Payloads (aka Registers) | 86.1 million (16%) | better than our goal of 65-68 million 🎉 |
| MTrie Vertices | 210 million (16%) | matches expected 2x-3x payload count reduction 🎉 |
| MTrie Size | 28.8 GB (7.2%) | better than our goal of 19-20 GB 🎉 |
| EN Checkpoint Size | 21.7 GB (6.2%) | better than our goal of 19-20 GB 🎉 |
| EN RAM Usage | TBD | never estimate (can be affected by other changes) |
| DB, caches, indexers, etc. | TBD | components handling payloads on AN, EN, etc. |

<details><summary> 🔍 Expand for more details.</summary>

|                  | Before | After | Reduction |
| ---------------------- |--------|-------|------------|
| Public Keys | 146,056,652 | 68,504,671 | 77,551,981 |
| Payloads (aka Registers)   |  539,650,919 | 453,516,554 | 86,134,365 |
| MTrie Vertices | 1,318,213,108 | 1,107,774,108 | 210,439,000 |
| MTrie Size (bytes) | 397,495,799,485 | 368,718,670,098 | 28,777,129,387 |
| EN Checkpoint Size (bytes) | 353,300,063,477 | 331,567,299,166 | 21,732,764,311 |
| EN RAM Usage | TBD | TBD | never estimate (can be affected by other changes) |
| DBs, caches, indexers, etc. | | | TBD on AN, EN, etc. |

EN state size reduction:
- before: 131821310896 + 270947341117 = 397495799485 bytes
- after: 110777410896 + 262372355730 = 368718670098 bytes

</details>

MTrie is the in-memory data structure containing the execution state. MTrie has vertices and payloads (atree and non-atree payloads).

This project only modifies non-atree payloads (account status and public key payloads).

Checkpoint files use a more space-efficient but slower encoding format to save MTrie to disk priodically, so checkpoints are smaller than MTrie.

If the Execution Nodes (Servers) RAM usage are reduced by less than the reduction in MTrie size, then it is likely that unrelated changes or activity is consuming extra RAM (vm configuration, OS page cache, db memory mapped files, updated components, etc.).  For example, some types of garbage collectors and databases might hold on to more RAM (to improve performance) if they detect more free RAM is available.
