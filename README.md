# draft-notes-about-deduplicating-public-keys

## Introduction

Each account is allowed to have multiple account keys.  An account key is the combination of a public key and associated data such as revoked status, weight, and sequence number.

Currently, when an account uses the same public key in multiple account keys, each account key stores its own copy of the same public key instead of using a shared immutable copy.

Some accounts have many thousands of account keys with their own copy of the same public key.  For example, one account on mainnet has just one unique public key and over 146K account keys storing their own full copy of the same public key.

Each of these duplicate public keys are currently stored in their own payload (aka register), which creates overhead for databases, indexers, execution state, and any other component that handle payloads on Access Nodes, Execution Nodes, etc.

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

MTrie is the data structure containing the execution state. MTrie has vertices and payloads (atree and non-atree payloads). The migration optimizes a subset of non-atree payloads.

If the EN RAM usage is reduced by less than the reduction in MTrie size, then it is likely that unrelated changes or activity is consuming extra RAM (vm configuration, OS page cache, db memory mapped files, updated components, etc.).

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
