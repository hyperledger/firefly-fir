- Feature Name: token_contracts
- Start Date: 2022-12-19
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)


# Summary
[summary]: #summary

FireFly token connectors need a more flexible way to support additional variants
of methods for mint, burn, transfer, and approve. Between the base ERC token standards,
the extensions defined by OpenZeppelin, and the extensions defined by FireFly, there
are quite a few common variants of these methods, and the current connector design is
very limited in how many variants it can support.

This is a work in progress and right now contains more questions than answers.
It will be updated as the proposed changes become clearer.

# Motivation
[motivation]: #motivation

In short, the current token connectors in the FireFly ecosystem cannot handle all
variants of token contracts that are likely to be encountered in real world projects.
In addition, adding support for new variants of mint/burn/transfers is currently too
difficult. Therefore, this proposal aims to provide an easier way to support additional
variants of token methods.

In long...

There are currently two official token connectors supported by the FireFly project -
firefly-tokens-erc20-erc721 and firefly-tokens-erc1155.
The former supports ERC20 for fungible tokens and ERC721 for non-fungible tokens,
while the latter supports ERC1155 for both.

FireFly requires a token connector to support 4 basic functions - mint, burn, transfer,
and approve. All of these token standards include an opinionated definition for
the "transfer" and "approve" family of methods, but do not include "mint" or "burn".
However, [OpenZeppelin](http://openzeppelin.com) has defined well-adopted extensions
for "mint" and "burn" on top of each of them. Although the current connectors
support all the required methods in some form, they're somewhat inconsistent about
following the OpenZeppelin conventions, or even (in some cases) the base standard.

The token connector TypeScript code requires ABI knowledge in order to make the proper
blockchain calls. Currently, each connector understands a few ABI variants of each method,
and leverages ERC165 in order to query the contract and guess which variant to use.
Below is the full set of ABI methods that are supported within the token connectors today.
The "withData" and "withURI" variants will be used if the contract advertises support for
them via ERC165; otherwise the plain variants will be used.

ERC20:
* `function mint(address to, uint256 amount)`
* `function mintWithData(address to, uint256 amount, bytes calldata data)`
* `function transferFrom(address from, address to, uint256 amount)`
* `function transferWithData(address from, address to, uint256 amount, bytes calldata data)`
* `function burn(address from, uint256 amount)`
* `function burnWithData(address from, uint256 amount, bytes calldata data)`
* `function approve(address spender, uint256 amount)`
* `function approveWithData(address spender, uint256 amount, bytes calldata data)`

ERC721:
* `function mint(address to, uint256 tokenId)`
* `function mintWithData(address to, uint256 tokenId, bytes calldata data)`
* `function mintWithURI(address to, uint256 tokenId, bytes calldata data, string memory tokenURI_)`
* `function safeTransferFrom(address from, address to, uint256 tokenId)`
* `function transferWithData(address from, address to, uint256 tokenId, bytes calldata data)`
* `function burn(address from, uint256 tokenId)`
* `function burnWithData(address from, uint256 tokenId, bytes calldata data)`
* `function approve(address to, uint256 tokenId)`
* `function setApprovalForAll(address operator, bool approved)`
* `function approveWithData(address to, uint256 tokenId, bytes calldata data)`
* `function setApprovalForAllWithData(address operator, bool approved, bytes calldata data)`

ERC1155:
* `function mintNonFungible(uint256 type_id, address[] calldata to, bytes calldata data)`
* `function mintNonFungibleWithURI(uint256 type_id, address[] calldata to, bytes calldata data, string memory _uri)`
* `function mintFungible(uint256 type_id, address[] calldata to, uint256[] calldata amounts, bytes calldata data)`
* `function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes memory data)`
* `function burn(address from, uint256 id, uint256 amount, bytes calldata data)`
* `function setApprovalForAllWithData(address operator, bool approved, bytes calldata data)`

While this list provides for a reasonable breadth of functionality (particularly when deploying a new
contract for your solution) it does not exhaustively serve all contract variants likely to be encountered.
In a few cases, it actually omits methods from the base standard. Many of the less-standard methods defined
by OpenZeppelin are also not supported.

These methods are part of the base standard and are not understood or utilized
by the current connectors:
* ERC20 transfer: `function transfer(address to, uint256 amount)`
* ERC721 transfer with data: `function safeTransferFrom(address from, address to, uint256 tokenId, bytes memory data)`

These methods are generated by the [OpenZeppelin wizard](https://wizard.openzeppelin.com)
and are not understood or utilized by the current connectors:
* ERC20 burn: `function burn(uint256 amount)`
* ERC20 burnFrom: `function burnFrom(address account, uint256 amount)`
* ERC721 mint: `function safeMint(address to, uint256 tokenId)`
* ERC721 auto-increment mint: `function safeMint(address to)`
* ERC721 burn: `function burn(uint256 tokenId)`
* ERC1155 mint: `function mint(address account, uint256 id, uint256 amount, bytes memory data)`
* ERC1155 mintBatch: `function mintBatch(address to, uint256[] memory ids, uint256[] memory amounts, bytes memory data)`
* ERC1155 burn: `function burn(address account, uint256 id, uint256 value)`
* ERC1155 burnBatch: `function burnBatch(address account, uint256[] memory ids, uint256[] memory values)`

Thus, to provide meaningful token functionality, all use cases currently require designing
a smart contract that 1) uses the signatures FireFly understands, and 2) includes ERC165 support.
Ideally, a contract written solely using methods from the base token standard and the OpenZeppelin
wizard would be usable with no special modifications at all.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A token connector can be provided up-front with details about the underlying
contract being used for a particular token pool.

Details TBD

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Once the connector has access to a contract ABI, it's easy for the connector to have a
prioritized mapping of methods that it looks for in order to perform a given operation.
For instance, ERC721 mint might prefer `mintWithData(address to, uint256 tokenId, bytes calldata data)`,
then `function safeMint(address to, uint256 tokenId)`, then `function mint(address to, uint256 tokenId)`,
etc. The first method found can be invoked.

The more complicated problem is how to deliver the ABI. Possible forms this could take:
1. Token connector can be configured with a custom set of ABIs at deploy time (vs a static set).
   Connector can somehow be told which ABI to use for each token pool, _or_ can introspect the contract
   to figure it out (closest to the current behavior - but needs to be something that works for all
   contracts, ie not dependent on ERC165).
2. FireFly passes an ABI to the token connector when a pool is created. The connector stores
   the ABI (would require adding storage, which we've avoided thus far) _or_ passes back a subset of
   the ABI to be passed back on future token calls.
3. FireFly passes an ABI to the token connector every time it invokes any token method (simple, but
   means a very large request body on every HTTP call).

For options 2-3, this probably implies that the user first creates a contract interface on FireFly,
then includes the interface ID (or name) at token pool creation time, so FireFly can know which ABI
to pass down to the connector.

# Drawbacks
[drawbacks]: #drawbacks

TBD

# Rationale and alternatives
[alternatives]: #alternatives

TBD

# Prior art
[prior-art]: #prior-art

TBD

# Testing
[testing]: #testing

TBD

# Dependencies
[dependencies]: #dependencies

TBD

# Unresolved questions
[unresolved]: #unresolved-questions

Lots.
