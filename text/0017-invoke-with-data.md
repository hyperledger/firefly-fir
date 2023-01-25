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
[Fabric](https://github.com/hyperledger/firefly/tree/main/smart_contracts/fabric/firefly-go), and it does
nothing on-chain other than emitting a `BatchPin` event which FireFly understands how to process.

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

## Application Contract

The application contract must fulfill certain requirements in order to leverage the new functionality.

First, it must know the on-chain location of the **FireFly Multiparty Contract**. How this is achieved
is out of scope for this FIR - it may be hard-coded in the application contract at deployment time, or it
may be set by invoking a method on the application contract. An application may leverage the fact that
this location is available by querying the FireFly `/status` API (under `multiparty.contract.location`
as of FireFly v1.1.0), and the application may leverage FireFly's `/invoke` APIs to invoke a method on
the application contract which sets the FireFly contract location. The application must also consider
how to update this location if a multiparty "network action" is used to migrate the network onto a new
FireFly multiparty contract.

Second, any external method of the contract that will be used for associating with off-chain payloads
must conform to [ERC5750](https://eips.ethereum.org/EIPS/eip-5750) - that is, the method signature must
end with a parameter for passing arbitrary unstructured data (in the case of Ethereum, the parameter
should be of type `bytes`, and for Fabric, a `string`).

Finally, the method(s) in question must invoke the new `pinBatchData` method of the
**FireFly Multiparty Contract** and pass along the data payload that was received in the final method
parameter. This should always be done as a final step before returning, after the method has performed
its own logic.

## FireFly Multiparty Contract

The multiparty contract is enhanced as part of this proposal, with a new external `pinBatchData` method.
This method takes a single packed "data" parameter. Specifically, this "data" parameter will be
expected to be an ABI-encoded `bytes` parameter for Ethereum, or a JSON-encoded `string` for Fabric.
The method will unpack this encoded value into a `struct` of 4 arguments (representing
`uuids, batchHash, payloadRef, contexts`), and then will pass those arguments to the existing `pinBatch`
method.

In the case of Ethereum, emission of the `BatchPin` event will be altered to emit `tx.origin` as the
`author`, instead of using `msg.sender` as it does currently.

TODO: add details around security concerns of `tx.origin` and recommended mitigations

## FireFly Core

The request body for all blockchain `/invoke` methods can now include a `message: {}` section (similar
to the request bodies for token transfers). When this is present, FireFly will prepare and seal a message
and return the ID as part of the 202 response (similar to what is done currently for token transfers).

The FireFly message will be marked (either with a new message `type` or some new field TBD) to indicate
that it must correlate with a contract invoke. This marking will also signal the batch assembler to treat
this message differently and to assemble it into a special "batch of one" rather than a normal batch.
This implies that batching of messages in FireFLy is not directly supported in this use case (although
the user could opt to batch their own data before passing it to FireFly).

The invoke operation will not actually be sent to the blockchain until the batch is prepared and
sealed. When it is ready, the 4 "batch pin" arguments (`uuids, batchHash, payloadRef, contexts`) will
be passed to the blockchain plugin along with the rest of the inputs for the invoke. The plugin will
inspect the method ABI to verify ERC5750 compliance, then will pack the 4 "batch pin" arguments in a
suitable manner (ABI-encoded `bytes` for Ethereum, or a JSON-encoded `string` for Fabric) and supply
them as the final argument to the method call.

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

Lots.
