---
layout: default
title: Token Approvals
nav_order: 3
---

- Feature Name: Token Approvals
- Start Date: 02/14/2022
- FIR PR:
- FireFly Component: tokens plugin, token connectors
- FireFly Issue:

# Summary
[summary]: #summary

This FIR introduces support for token approvals within FireFly. This includes changes to the `fftokens` plugin and new set of API's to submit and query the state of existing approvals.

# Motivation
[motivation]: #motivation

Permitting some entity (i.e. a smart contract or another user) to perform token transfers on your behalf is a common pattern within the Ethereum token standards. Adding support for token approvals significantly expands the types of use cases FireFly can support. The goal of this FIR is to enhance FireFly's current token feature set with a new, simple API to manage token approvals.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This FIR extends the core tokens functionality by adding support for approvals with a new set of API's:

   | Method | Endpoint                                    | Description                                  |
   | ------ | ------------------------------------------- | -------------------------------------------- |
   | `POST` | `/namespaces/{ns}/tokens/approvals`         | Submit a new approval                        |
   | `GET`  | `/namespaces/{ns}/tokens/approvals`         | Query all approvals                          |
   | `GET`  | `/namespaces/{ns}/tokens/approvals/:id`     | Query query approvals by ID                  |

Similar to mint, burns, and transfers, an approval must be tied to a `pool`. This works great for ERC20 and ERC721, because there is one pool per connector. ERC1155 is unique for two reasons: it allows multiple token pools and the OpenZeppelin standard only supports `setApprovalForAll`. This means when submitting an approval on a token pool when using the ERC1155 connector, that approval will be propagated to **all** other pools.

### Database Updates
Adds a new `tokenapproval` table
```
  seq              INTEGER         PRIMARY KEY AUTOINCREMENT,
  local_id         UUID            NOT NULL,
  pool_id          VARCHAR(1024)   NOT NULL,
  key              VARCHAR(1024)   NOT NULL,
  operator_key     VARCHAR(1024)   NOT NULL,
  approved         BOOLEAN         NOT NULL,
  protocol_id      VARCHAR(1024)   NOT NULL,
  tx_type          VARCHAR(64),
  connector        VARCHAR(64),
  namespace        VARCHAR(64),
  info             TEXT,
  tx_id            UUID,
  blockchain_event UUID,
  created          BIGINT          NOT NULL
```

### Protocol ID's
Protocol ID's are a connector specific identifier that is passed into FireFly core with the proxied ethconnect event. The `token-approval` Protocol ID format is: `<owner>:<operator>`.

### ERC1155 Connector Changes

The FireFly ERC1155 connector has been updated to support submitting `setApprovalForAll` transactions and receiving `ApprovalForAll` events from ethconnect. A wrapper method `setApprovalForAllWithData` has been created with the extra parameter `data` so extra information can be included in the subsequent events' `inputArgs`. Currently, `data` is used to pass the FireFly transaction id with the event. Once a `ApprovalForAll` event is received, the connector proxies a `token-approval` event to FireFly with the outputs: id, poolId, signer, operator, approved, and data.

When is an approval is submitted for a token pool, it will be propagated to all other pools. This is achieved by leveraging ethconnect subscriptions within the connector. Each time a token pool is activated, ethconnect subscriptions are created for the events `TransferSingle`, `TokenCreate`, `TransferBatch`, and now `ApprovalForAll`. Each time an `ApprovalForAll` event is emitted from ethconnect, regardless of which pool it was intended for, every pools' approval subscription will receive that event and pass it to FireFly. This is how approval events are propagated and recorded for every pool.

Normally, the ERC1155 connector allows the block number from which the subscription starts to be configurable. In the case of `ApprovalForAll`, the subscription must start from block `0` so no approvals are missed by FireFly.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

No further information is needed.

# Drawbacks
[drawbacks]: #drawbacks

There is little risk associated with this feature. No existing functionality has changed.


# Rationale and alternatives
[alternatives]: #alternatives

TBD

# Prior art
[prior-art]: #prior-art

N/A

# Testing
[testing]: #testing

Existing unit tests and e2e tests will be sufficient to test the new feature, after being adapted to work with the new approach.

# Dependencies
[dependencies]: #dependencies

N/A

# Unresolved questions
[unresolved]: #unresolved-questions

Support for ERC20 and ERC721 connectors is not included in this FIR and will be addressed at a later time. This design was intended to be sufficient for all existing connectors, but no thorough verification has occured.
