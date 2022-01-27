---
layout: default
title: FIR Template
nav_order: 3
---

- Feature Name: Transaction Rework
- Start Date: 2022-01-27
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

FireFly contains a construct called a `Transaction`, but the exact purpose of this item has been
poorly defined. As a result, it has come to serve multiple purposes in the system, but all of them
incompletely.

The goal of this improvement is to clearly define the scope and purpose of a `Transaction` going
forward.

# Motivation
[motivation]: #motivation

The original requirements for what defines a FireFly `Transaction` (as of v0.11.5) were:
- Provides a container for grouping related async operations together
- Provides an overall "status" of pending/successful/failed
- Corresponds to exactly 1 blockchain transaction, which emits exactly 1 event
- Records interesting data from the blockchain, such as signer, block number, and transaction hash

However, this definition has become problematic as the system evolves:
- It's impossible or difficult to represent a group of operations that triggers no blockchain transaction,
  or that triggers multiple blockchain transactions, or a blockchain transaction with multiple events
- The "status" field is inconsistently updated, such that it's almost always useless and requires
  separate queries to other APIs to determine if a transaction has succeeded or not
- The `BlockchainEvent` type introduced in FIR #2 has become a better home for interesting blockchain
  data, and now overlaps with some of what is recorded in `Transaction`

It's clear that some cleanup is needed to resolve these problems and allow `Transaction` to cover
future needs as new use cases are added.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A FireFly `Transaction` is a fundamental unit of work in the system. It is first of all a container for:
- One or more related `Operations`, which may trigger work via on- or off-chain plugins
- Zero or more `BlockchainEvents`, which record blockchain feedback
- Any new FireFly object that is generated as a result of the work, such as a `Batch` or `TokenTransfer`

Additionally, a FireFly `Transaction` frequently maps 1:1 to a blockchain transaction (though it
may map to zero or many). In all cases, it stores a list of identifiers for the related blockchain
transaction(s), for easy mapping such as when cross-referencing with a block explorer.

Finally, a FireFly `Transaction` exposes a meaningful status, which is computed on demand and takes
into account the status of all contained `Operations`, `BlockchainEvents`, etc to provide an
accurate and useful assessment as to whether the work was completed as expected.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Data model changes

The full set of fields on the `Transaction` object will be:
- ID - unique, FireFly-generated UUID
- Namespace - FireFly namespace
- Type - enum representing the type of transaction
- Created - timestamp that the object was created in FireFly
- BlockchainIDs - string array of blockchain identifiers (such as Ethereum transaction hashes)

This represents the _deletion_ of the following fields:
- Signer - not needed, as it is already recorded in other relevant child objects
- Reference - redundant, as child objects all contain a back-reference to `Transaction`
- Hash - not needed, as `Transaction` now contains no sensitive/verifiable data
- Status - now computed on demand as a separate API call
- ProtocolID - replaced by BlockchainIDs
- Info - moved to  a child `BlockchainEvent` object

## Types of transactions

There are 3 types of `Transaction` as of v0.11.5, with another being added currently in support of
FIR #2 (which is intertwined with this work):
- `TransactionTypeBatchPin`
  - on initiator, contains some operations of types `OpTypeBlockchainBatchPin`,
    `OpTypePublicStorageBatchBroadcast`, `OpTypeDataExchangeBatchSend`, and/or `OpTypeDataExchangeBlobSend`
  - receives one `BlockchainEvent` for a "batch pin" event
  - generates a FireFly `Batch`
- `TransactionTypeTokenPool`
  - on initiator, contains exactly two operations, of type `OpTypeTokenCreatePool` and `OpTypeTokenAnnouncePool`
  - receives one `BlockchainEvent` for a "token pool" event
  - generates a FireFly `TokenPool`
- `TransactionTypeTokenTransfer`
  - on initiator, contains exactly one operation of type `OpTypeTokenTransfer`
  - receives one or more `BlockchainEvents` for "token transfer" events
  - generates one or more FireFly `TokenTransfers`
- `TransactionTypeContractInvoke`
  - on initiator, contains exactly one operation of type `OpTypeContractInvoke`
  - does not receive any events or generate any objects
  - is not shared with other nodes

As a future work item, unpinned messages can also be grouped under a new `TransactionTypeUnpinned`. This
is not part of the implementation here, but should be supported by this model.

## API changes

A new `/namespaces/{ns}/transactions/{txid}/status` API will be added, for checking the status of a transaction.
This will compute the status in a way that is particular to each type of transaction, but roughly follows these
guidelines:
- If this is the initiating node, and any child `Operation` has failed, the transaction hass failed
- If this is the initiating node, and any child `Operation` is pending, the transaction is pending
- If a `BlockchainEvent` is expected but not yet received, the transaction is pending
- If the expected `BlockchainEvents` are received, any `Operations` are successful, _and_ the target object is
  generated, the transaction has succeeded

The status response object will include a top-level `status` of "Pending", "Succeeded", or "Failed", and a
`details` array with information on each object checked to build the status.

A new `/namespaces/{ns}/transactions/{txid}/blockchainevents` API will be added for conveniently fetching the
`BlockchainEvents` received in regards to a particular `Transaction`.

# Drawbacks
[drawbacks]: #drawbacks

This is a breaking API change, because many fields are removed from `Transaction` or moved to
other objects. Any applications querying and relying on these removed fields will need to be
redesigned to retrieve the information from elsewhere.

# Rationale and alternatives
[alternatives]: #alternatives

Due to the very specific meaning attached to the word "transaction" in the blockchain space,
significant thought was devoted to whether we should abandon the word altogether and use a different
word for this grouping construct. In the end, the term has been kept because a FireFly `Transaction`:
- does accurately align with the general definition of a transaction in the computing world
  (an input message to a computer system that must be dealt with as a single unit of work)
- does usually correspond 1:1 with a blockchain transaction - and the instances where it does not
  can be a positive opportunity to demonstrate the project's on- and off-chain flexibility

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus, global state, transaction processors, and smart contracts
  implementation proposals: Does this feature exists in other distributed
  ledgers and what experience have their communities had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other distributed ledgers, provide readers of your FIR with
a fuller picture.  If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if it is an adaptation.

Note that while precedent set by other distributed ledgers is some motivation,
it does not on its own motivate an FIR.  Please also take into consideration
that FireFly sometimes intentionally diverges from common distributed
ledger/blockchain features.

# Testing
[testing]: #testing

- What kinds of test development and execution will be required in order
to validate this proposal, beyond the usual mandatory unit tests?
- List integration test scenarios which will outline correctness of proposed functionality.

# Dependencies
[dependencies]: #dependencies

- Describe all dependencies that this proposal might have on other FIRs, known JIRA issues,
Hyperledger FireFly components.  Dependencies upon FIRs or issues should be recorded as 
links in the proposals issue itself.

- List down related FIRs proposals that depend upon current FIR, and likewise make sure 
they are also linked to this FIR.

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the FIR process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this FIR that could be
  addressed in the future independently of the solution that comes out of this
  FIR?
