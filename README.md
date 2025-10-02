# draft-notes-about-deduplicating-public-keys

## Introduction

Each account is allowed to have multiple account keys.  An account key is the combination of a public key and associated data such as revoked status, weight, and sequence number.

Currently, when an account uses the same public key in multiple account keys, each account key stores its own copy of the same public key instead of using a shared immutable copy.

Some accounts have many thousands of account keys with their own copy of the same public key.  For example, one account on mainnet has just one unique public key and over 146K account keys storing their own full copy of the same public key.

Each of these duplicate public keys are currently stored in their own payload (aka register), which creates overhead for databases, indexers, execution state, and any other component that handle payloads on Access Nodes, Execution Nodes, etc.

## Stats

In progress...

## Goals

Initial goals:
- remove ~46% of all public keys stored
- reduce memory and storage used by ~19 GB
- reduce registers stored by ~65M

Updated goals based on newer mainnet data and clarification about RAM:
- remove ~48% of all public keys stored
- reduce state size and disk used by ~20 GB each
- reduce number of registers by ~68 million
- reduce RAM used by EN is TBD after deployment to mainnet.  Memory use on Execution Nodes can sometimes reduce by ~3x state size reduction but that isn't guaranteed due to external factors like Go garbage collection, unrelated components/processes, OS page cache, etc.

## Results

In progress... (so far, all results are better than the stated goals).

There were ~77 million duplicate payloads in the older mainnet snapshot, and migration reduced payload count by ~86 million payloads. (TODO: update this using latest Oct 2025 mainnet snapshot)

Most accounts on mainnet only have 1 account key but they still use fewer bytes in the new data format.

## Design and Implementation

- ### Design of New Data Format

- ### Efficiently Detecting Duplicates At Runtime

- ### RLP++ Encoding (I don't know if I'm the first person to invent this encoding)

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
