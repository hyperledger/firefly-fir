---
layout: default
title: Custom NFT URI's
nav_order: 13
---

- Feature Name: custom_nft_uri
- Start Date: 2022-06-21
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

It's common for non-fungible tokens to have an associated URI which points to an immutable and unique identifier for a particular digital, a common example being an IPFS CID. FireFly does not explicitly support this in both the ERC721 and ERC1155 connectors. To support more use cases, both FireFly and its token connectors will be updated to allow setting a custom URI when minting an NFT. This behavior will be optional and will not deprecate existing functionality.

# Motivation
[motivation]: #motivation

NFT URI's commonly point to the metadata schema (outlined in EIP 721) stored in IPFS or somewhere similar. Currently, FireFly does not support this behavior in either of the reference implementation connectors for non-fungible tokens (ERC-721 and ERC-1155). As FireFly aims to support more blockchain ecosystems, supporting this usecase seems important.

This FIR represents enhancements to both FireFly core and the existing implementations of the ERC-721 and ERC-115 token connectors to support custom NFT URI's:
 * The FireFly core `/mint` API now supports an optional `uri` field  for non-fungible tokens
 * Updates to both ERC-1155 and ERC-721 smart contracts to support custom URI's

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in FireFly and you were
teaching it to another FireFly developer. That generally means:

URI's in non-fungible tokens are commonly used to point to metadata regarding that particular NFT. EIP-721 defines a schema for what the metadata should look like, but where that metadata lives is arbitrary. Within FireFly, it's very possible that data could be stored within Data Exchange or IPFS and the DX Hash or IPFS CID is the identifier stored as the URI. 

The FireFly API for minting tokens supports an optional `uri` parameter, where the value will be passed to the configured token connector to be used as the NFT URI. Both ERC-721 and ERC-1155 smart contracts expose a public `MintWithURI` method, when invoked will override the existing `baseURI` parameter on the token.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

# Drawbacks
[drawbacks]: #drawbacks

One potential drawback is introducing more restrictions on NFT URI's. Similar to OpenZeppelin, we previously did not impose any restrictions on how setting NFT URI's should be implemented. This FIR will change that.

# Rationale and alternatives
[alternatives]: #alternatives

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Testing
[testing]: #testing

Migration testing of existing environments of various configs will be critical to avoid
breaking things as a result of this change.

New E2E tests for nodes with different combinations of namespace configs will also need to be added.

# Dependencies
[dependencies]: #dependencies

- TBD

# Unresolved questions
[unresolved]: #unresolved-questions

- TBD