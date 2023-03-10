- Feature Name: invoke_with_data
- Start Date: 2023-01-24
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary

[summary]: #summary

This proposal details a way to tie any arbitrary piece(s) of off-chain data to any piece of on-chain
logic, within a single blockchain transaction, allowing FireFly to perform aggregation and emit an
event only when both pieces (the off-chain data payload and confirmation of the on-chain transaction)
have been received.

# Motivation

[motivation]: #motivation

FireFly currently provides the ability to "pin" an off-chain message to the blockchain by writing a hash
and ordering context into a specific `pinBatch` transaction on the FireFly multiparty contract.
Reference implementations of this contract are provided for
[Ethereum](https://github.com/hyperledger/firefly/tree/main/smart_contracts/ethereum/solidity_firefly) and
[Fabric](https://github.com/hyperledger/firefly/tree/main/smart_contracts/fabric/firefly-go). The
contract simply emits a `BatchPin` event, and FireFly will correlate the availability of the off-chain
data (received via IPFS or private send) with the arrival of the `BatchPin` event, such that
subsequent processing on the same topic is not triggered until _both_ the off-chain data and the on-chain
event have been received.

Token operations such as transfers and approvals can also be associated with an off-chain message, by
writing a hash of the message into an extra parameter during the on-chain token transaction. In this case,
a separate `pinBatch` transaction is triggered by FireFly after the token operation succeeds, so that the
event from that transaction can be used for sequencing purposes. This allows FireFly to aggregate the
token event and the message (although it ends up requiring two separate blockchain transactions).

While these specific cases are helpful in binding certain on- and off-chain events, more advanced
blockchain use cases demand a way to bind an off-chain payload to _any_ arbitrary blockchain transaction.
This proposal lays out a way to achieve that goal, by adding new enhancements to FireFly and by defining
a set of best practices for compliant smart contracts.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

TBD

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

There are 3 components that must interact to enable this feature: **FireFly Core**, the
**FireFly Multiparty Contract** (which provides `pinBatch`), and the **Application Contract** (which
defines the custom blockchain logic for a given application).

Most requirements are universal, but some are naturally blockchain-specific. These differences are
called out for the two currently supported blockchains (Ethereum and Fabric) below as appropriate.

## FireFly Core

The request body for all blockchain `/invoke` methods can now include a `message: {}` section (similar
to the request bodies for token transfers). When this is present, FireFly will prepare and seal a message
and return the ID as part of the 202 response (similar to what is done currently for token transfers).

FireFly will also generate a transaction with a new type of `contract_invoke_pin`. The ID of this
transaction will be stored on the message object (but not hashed as part of the message header).
When processing messages of this type, the batch assembler will ensure that they are _not_ batched,
but that the message is assembled into a special "batch of one". This implies that batching of messages
in FireFly is not directly supported in this use case (although the user could opt to batch their own
data before passing it to FireFly). The batch will also inherit the transaction type and ID from the
message, rather than generating a new transaction.

The invoke operation will not actually be sent to the blockchain until the batch is prepared and
sealed. When it is ready, the "batch pin" arguments (`uuids, batchHash, payloadRef, contexts`) will
be passed to the blockchain plugin along with the rest of the inputs for the invoke. The plugin will
inspect the method schema to verify it can be used (ie checking that it complies with ERC5750 or some
interpretation appropriate to the blockchain in question) - and fail the request if there are any
problems. Then it will pack the "batch pin" arguments in a suitable manner (i.e. ABI encoding for
Ethereum, or JSON encoding for Fabric) and supply them as the final argument to the method call.

## FireFly Multiparty Contract

### Ethereum

- The multiparty contract is enhanced with a new external `pinBatchData` method.
  This method takes a single packed "data" parameter of type `bytes`. This will be able to receive
  an ABI-encoded `bytes` value that will unpack into a `struct` of 4 arguments (representing
  `uuids, batchHash, payloadRef, contexts`). The method will emit a `BatchPin` event similar to the
  existing `pinBatch` method.
- Emission of the `BatchPin` event will also be altered to emit `tx.origin` as the
  `author`, instead of using `msg.sender` as it does currently.

> Usage of `tx.origin` in the emitted events means that if other smart contracts are allowed to call
> `pinBatch` on the FireFly Multiparty Contract, the event will record the original calling user as the
> author of the batch. This is specifically a desirable change in the context of how this contract is
> expected to be used. However, users should consider if there is any risk of malicious contracts
> leveraging this to advertise a batch as a side-effect of some unrelated transaction, without the calling
> user's consent. If needed, the FireFly Multiparty Contract should be customized for an individual
> use case by limiting the parties or contracts that are allowed to call into it.

### Fabric

- No changes are required to the multiparty contract, since it is not used when a batch pin is
  performed by a custom contract (see below).

## Application Contract

Any external method of the contract that will be used for associating with off-chain payloads
must provide an extra parameter for passing the encoded batch data. This must be the last parameter
in the method signature. This convention is chosen partly to align with the Ethereum
[ERC5750](https://eips.ethereum.org/EIPS/eip-5750) standard, but should serve as a straightforward
guideline for nearly any blockchain.

This method must emit a `BatchPin` event that can be received and parsed by FireFly. Exactly how
the data is unpacked and used to emit this event will differ for each blockchain.

### Ethereum

- The method in question will receive packed "batch pin" data in its last method parameter (in the
  form of ABI-encoded `bytes`). The method must invoke the new `pinBatchData` method of the
  **FireFly Multiparty Contract** and pass along this data payload. It is generally good practice to
  trigger this as a final step before returning, after the method has performed its own logic.
- This also implies that the contract must know the on-chain location of the
  **FireFly Multiparty Contract**. How this is achieved is out of scope for this FIR - it may be hard-coded
  in the application contract at deployment time, or it may be set by invoking a method on the application
  contract. An application may leverage the fact that this location is available by querying the FireFly
  `/status` API (under `multiparty.contract.location` as of FireFly v1.1.0), and the application may
  leverage FireFly's `/invoke` APIs to invoke a method on the application contract which sets the FireFly
  contract location. The application must also consider how to update this location if a multiparty
  "network action" is used to migrate the network onto a new FireFly multiparty contract.

### Fabric

- The method in question will received packed "batch pin" data in its last method parameter (in the
  form of a JSON-encoded `string`). The method must unpack this argument into a JSON object.
- The contract must directly set a `BatchPin` event in the same format that is used by the
  **FireFly Multiparty Contract**.
- FireFly must also be configured to listen for `BatchPin` events from _any_ chaincode on the channel
  (not just the specific "firefly" chaincode). This is done by enabling a new
  `namespaces.predefined[].multiparty.contract[].options.globalListener` flag in the FireFly config.
  Note that this must be enabled before starting FireFly for the first time in order to take effect.

# Drawbacks

[drawbacks]: #drawbacks

TODO

# Rationale and alternatives

[alternatives]: #alternatives

TODO

# Prior art

[prior-art]: #prior-art

TODO

# Testing

[testing]: #testing

TODO

# Dependencies

[dependencies]: #dependencies

TODO

# Unresolved questions

[unresolved]: #unresolved-questions

TODO
