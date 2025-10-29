# draft-notes-about-deduplicating-public-keys

## How We Reduced Payload Count to Cut 211M Hashes and 29 GB of State

By optimizing how we store account keys, we reduced execution state size by 28.8 GB (over 7 percent of the entire execution state). Perhaps more importantly, we did this by reducing the number of payloads, and we also reduced the payload count's growth rate.

Reducing the number of payloads matters because each payload adds different overhead (cpu, ram, disk, network) to different types of servers running databases, caches, execution state, indexers, etc. that handle payloads.

As just one example, MTrie (the execution state) incurs 192-288 bytes of overhead for each 72-78 byte payload that held an account key.  The overhead is larger than the payload size because each payload requires about 2-3 MTrie vertices (96 bytes each).  We reduced enough payloads for us to no longer need 210 million cryptographic hashes and the MTrie vertices containing them.

To put what we achieved into perspective, we found 77.6 million payloads with duplicate keys, but our improved data format reduces the number of payloads by 86.1 million without using compression (not a typo)!

These incremental improvements add up to have a cumulative impact on operational costs, uptime, and performance.  Although payload count increases daily, Execution Nodes, etc. can handle more payloads now compared to 9 months ago, without adding extra RAM.

Beyond Execution Nodes, reducing payload counts and their overhead helps reduce hardware costs and energy consumption on various servers that handle payloads.  By improving resource utilization, we gain headroom to handle increased spikes in demand, reduce downtime, and reduce costs.  Memory use reduction on AN, EN, etc. are hard to measure since we are deploying unrelated projects at the same time, but we know we are loading 28.8 GB less execution state into RAM on EN, and memory usage reduction on EN in a prior migration project was reported to be a multiple of the state size reduction.

UPDATE: Memory use reduction is still about -150 GB less RAM on each EN server about 7 days after the Oct. 22, 2025 mainnet spork.  This is over 5x MTrie size reduction! 🎉

At the same time, the change does not take anything away from end users. [...]

## Scope

This project optimizes non-atree payloads to reduce the total number of payloads.  Specifically, this project only modifies a subset of non-atree payloads (account status and public key).

Migration optimizes the existing payloads during spork.  Accounts that only have 1 account key (most mainnet accounts) cannot have duplicates, but they use a few bytes less in the new data format.

Runtime optimizes new payloads by detecting duplicate pubic keys being added, and will store them in the new deduplicated format.  Unlike migration, runtime duplicate key detection is not 100% as a tradeoff for faster speed and more efficient storage overall.

NOTE:  To avoid sacrificing speed, this project does not use compression libraries (e.g., LZ4, zstd, etc.).

## Goals

Our goal is to improve efficiency of databases, caches, indexers, MTrie, etc. that handle payloads on various nodes by reducing the number of payloads.

Reducing the number of payloads also potentially benefits 3rd-party servers if they handle payloads.

By improving resource utilization and gaining headroom to handle spikes in demand, etc. we can reduce risk of downtime and reduce operational costs.

This project adds to the cumulative impact of reducing payload counts.  This project exceeded each of our quantifiable goals.

<details><summary> 🔍 Expand for more detailed goals</summary>

#### Initial goals:
- remove ~46% of all public keys stored
- reduce memory and storage used by ~19 GB
- reduce registers stored by ~65M

#### Updated goals based on newer mainnet data and clarification about RAM:
- remove ~48% of all public keys stored
- reduce state size and disk used by ~20 GB each
- reduce number of registers by ~68 million
- reduce RAM used by EN is TBD after deployment to mainnet.  Memory use on Execution Nodes can sometimes reduce by ~3x state size reduction but that isn't guaranteed due to external factors like Go garbage collection, unrelated components/processes, OS page cache, etc.

</details>

## Actual Results (Oct. 22, 2025 Mainnet Spork)

|                  | Reduction | Notes |
| ---------------------- |--------| --- |
| Public Keys | 77.6 million (53.1%) | better than our goal of 46-48% 🎉 |
| Payloads (aka Registers) | 86.1 million (16%) | better than our goal of 65-68 million 🎉 |
| Payload Size | 8.9 GB | not estimated but better than private expectations 🎉 |
| MTrie Vertices | 210 million (16%) | matches expected 2x-3x payload count reduction 🎉 |
| MTrie Size | 28.8 GB (7.2%) | better than our goal of 19-20 GB 🎉 |
| EN Checkpoint Size | 21.7 GB (6.2%) | better than our goal of 19-20 GB 🎉 |
| EN RAM Usage | ~150 GB | not estimated but better than private expectations 🎉 |
| DB, caches, indexers, etc. | TBD | components handling payloads on AN, EN, etc. |

The ~150 GB RAM reduction on each EN server (7 days after spork as of today) exceeded expectations.

Speedup is also better than expected, but multiple changes were deployed at the same time, so it is hard to separate.

<details><summary> 🔍 Expand for details</summary>

#### Mainnet Spork (Oct. 22, 2025)

NOTE: After spork, the reverse migration test result (new -> old) matched the regular migration (old -> new) result.

MTrie size reduction (bytes):
- payloads: 8587120345
- vertices: 20226276288
- total: 28813396633 (28.8 GB)

Checkpoint file size:
- old format: 354914430454 bytes (created by reverse migration from new -> old for comparison)
- new format: 333167435599 bytes
- reduction: 21746994855 bytes (27.1 GB)

MTrie node (aka vertex) count:
- before: 1323678476
- after: 1112988098
- count reduction: 210690378 (211M cryptographic hashes!)
- size reduction (count * 96 bytes): 20226276288

Payload count:
- before: 541888998
- after: 455650954
- reduction: 86238044

Payload size:
- before: 272233935818
- after: 263646815473
- reduction (bytes): 8587120345

</details>

### Migration Speed

The key deduplication part of the migration only took 5m36s using `nworkers=10` on m1 (within the longer running common migration framework of loading/saving checkpoints, etc. which unchanged from prior migrations).  The speed of migration in pre-spork tests using `nworkers=64` allowed us to use m1 instead of m3 server for the mainnet spork on Oct. 22, 2025 🎉.

## Results Based on October 1, 2025 Mainnet Checkpoint

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

If the Execution Nodes (Servers) RAM usage are reduced by less than the reduction in MTrie size, then it is likely that unrelated changes or activity is consuming extra RAM (vm configuration, OS page cache, db memory mapped files, updated components, etc.).  For example, some types of garbage collectors and databases might hold on to more RAM (to improve performance) if they detect more free RAM is available.

## Design and Implementation

I designed the new data format to reduce payload count by more than the total number of duplicate payloads.

Additionally, I wanted to make the new data format support efficient runtime duplicate key detection (when new keys are added by accounts).

And since most accounts only have 1 account key (no duplicate public keys possible), the design special cases those accounts to avoid adding overhead.

### Design of New Data Format

### Efficiently Detecting Duplicates At Runtime

### Encoding Formats

The new data format uses:
- RLP encoding (for the data already encoded in RLP in the old format)
- Raw bytes (for very simple new data where using a codec would be overkill)
- RLE++ (for efficiently encoding both repeating values and non-repeating values)

### RLE++ Encoding (I don't know if I'm the first person to invent this encoding)

The new data format uses a new encoding called RLE++ to efficiently encode both repeating values and non-repeating values in the same encoding.

I named it RLE++ because it adds a feature to RLE ([run-length encoding](https://en.wikipedia.org/wiki/Run-length_encoding)) that supports efficient encoding of non-repeating values created by using the ++ operator (incrementing values).

## Results of Optimized Migration Speed

## Results of Optimized Runtime Storage to Reduce Need for Future Migrations

## Work Done

This list is not exhaustive and it doesn't show time spent on design, getting familiar with existing codebase, optimizing migration speed before opening first PR, or extra testing (beyond unit tests and integration tests).

- https://github.com/onflow/flow-go/pull/7738 Add account public key deduplication migration
- https://github.com/onflow/flow-go/pull/7829 Add runtime public key duplicate detection and storage with support for account status v4 format
- https://github.com/onflow/flow-go/pull/7834 Fix the RegisterSize() function
- https://github.com/onflow/flow-go/pull/7835 Change report format for storage used migration from JSON to CSV
- https://github.com/onflow/flow-go/pull/7841 Reduce register reads by using Accounts.GetRuntimeAccountPublicKey
- https://github.com/onflow/flow-go/pull/7875 Create diff-keys utility to compare account public keys before and after migration
- https://github.com/onflow/flow-go/pull/7878 Add option to validate account public key migration
- https://github.com/onflow/flow-go/pull/7878 Support account status v4 in checkpoint-collect-stats util

## Migration and Runtime Tests

### Beyond Unit Tests and Integration Tests

Tests passed using older mainnet and testnet snapshots, but I test again when I need to get newer snapshots for something else. For example, here are some tests running last night using Oct 1, 2025 snapshot:

<img width="3833" height="780" alt="image" src="https://github.com/user-attachments/assets/e3dc931d-6d95-4bb9-b0ce-c26843ec9cbf" />

### Tests Used All Account Keys From Mainnet and Testnet

From a storage and retrieval perpective, I tested [PR 7829](https://github.com/onflow/flow-go/pull/7829) for runtime public key handling with the same level of testing I used for the migration ([PR #7738](https://github.com/onflow/flow-go/pull/7738)).  I essentially round-trip tested every account key from both mainnet and testnet to get the expected results.

During runtime tests, every account key from a mainnet and testnet snapshot were individually added and the stored results were compared to each corresponding account key that was stored in the original mainnet and testnet snapshot.

These extra tests confirmed the deduplicated public keys provide access to all the same account key data that existed prior to deduplication, such as the individual weight and sequence number of each previously duplicate key.
