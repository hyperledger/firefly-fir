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

A user of FireFly may first create a contract interface (FFI) for their token
contract. When creating a token pool, the interface may be specified (by ID or
by name and version), and FireFly will include details on the interface during
the pool creation flow with the connector.

A token connector supporting this flow will inspect the interface, and will return
a body to FireFly specifying any methods it may want to utilize
for the 4 core token APIs: `mint`, `burn`, `transfer`, and `approval`.

FireFly will store this listing of methods on the token pool, and
will include it when broadcasting the pool definition to other nodes.

When FireFly is asked to perform a `mint`, `burn`, `transfer`, or `approval` in
this pool, it will look up the relevant methods and include them in the request
to the connector.

Thus the connector should have a useful subset of the interface methods at its
disposal for every operation. It can choose which one to call, and pass the method
definition along to the blockchain connector.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

FireFly's `/tokens/pools` POST gets a new section:
```
"interface": {
   "id": "",
   "name": "",
   "version": "",
}
```

Either "id" or "name" and "version" must be specified.

The creation request from FireFly to the token connector may include a new field:
```
"interface": "<id of interface>"
```

If "interface" is undefined, the connector will fall back to the original method
of inspecting/guessing which contract interface is in use. If "interface" is
defined, it signals that the connector will be provided with interface details
later. Note: while this could technically be a boolean field, defined/undefined
feels clearer in case the connector would like to record the interface ID somehow.

The (synchronous or asynchronous) pool creation message from the connector
will include a new field as well, to specify the interface format this pool understands.
This will generally be hard-coded in the connector source (similar to how "standard"
is hardcoded to something like "ERC721"), and it must be either "abi" or "ffi":
```
"interfaceFormat": "abi"
```

The token connector exposes a new API `/checkinterface`. After successful pool
creation, FireFly may invoke this API with a request of the form (dependent on
which interface format the connector understands):
```
{
  "abi": [ /* methods in ABI format */ ]
}

or

{
   /* FFI definition */
}
```

The connector responds synchronously with a 200 and a body of the form:
```
{
  "mint": { /* subset of interface */ },
  "burn": { /* subset of interface */ },
  "transfer": { /* subset of interface */ },
  "approval": { /* subset of interface */ }
}
```

Internally, the connector defines a map of all the possible method signatures it supports for each
API operation (mint/burn/transfer/approval). Each variant is prioritized and includes a DTO mapping
function to map parameters into the correct positions.

When asked by FireFly to check an interface, the connector matches the provided interface against
the signature map and determines all methods that it understands in order to return them to FireFly.

Each of the token connector's APIs `mint`, `burn`, `transfer`, and `approval` may now receive an
additional field, where FireFly should pass back the subset of the interface definition that
matches the API being invoked:
```
{
  "interface": { /* subset of interface provided above */ }
}
```

At the time an operation is invoked and FireFly provides the FFI/ABI for the needed methods,
the connector references the signature map,  finds the first/best method that can satisfy the
operation, uses the corresponding mapping function to map the DTO into a list of parameters,
then passes the method definition and parameter list on to the blockchain connector.

This means it's possible (for instance) for an ERC721 connector to prefer
`mintWithData(address, uint256, bytes)`, then `safeMint(address, uint256)`, then
`mint(address, uint256)`, etc. The first method supported on a given contract will be invoked.

# Drawbacks
[drawbacks]: #drawbacks

This pushes more responsibility for knowledge of the token contract out of the connector and back to
both FireFly and the application developer. The split of responsibilities is fuzzier, as neither the
developer, nor FireFly, nor the token connector fully "owns" the responsibility of deciding how and
which blockchain methods to call for token operations. The decision is a more complicated back-and-forth
process, with different pieces of knowledge being held by each party.

Assuming we continue to support the original/simpler behavior of token connectors, wherein they perform
a very simple contract inspection without help from FireFly, this complicates the pool creation flow
(which is already one of the more complex flows in FireFly) with even more conditional branches.

# Rationale and alternatives
[alternatives]: #alternatives

The most complicated problem in this proposal is how and when to deliver the interface from FireFly to the
token connector. Possible alternatives to the current proposal that have been considered:

1. Token connector can be configured with a custom set of ABIs at deploy time (vs a static set).
   Connector can somehow be told which ABI to use for each token pool, _or_ can introspect the contract
   to figure it out (closest to the current behavior - but needs to be something that works for all
   contracts, ie not dependent on ERC165).
2. FireFly passes a full ABI to the token connector when a pool is created. The connector stores
   the ABI and references it later as needed (would require adding storage, which we've avoided thus far).
3. FireFly passes a full ABI to the token connector every time it invokes any token method (simple, but
   means a very large request body on every HTTP call).

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

Should there be a way to trigger FireFly to call `/checkinterface` again (such as after
upgrading a token connector) in order to re-initialize the list of methods supported
for each operation?

Can we allow a user to inject supported method signatures via API without changing the
signature map in the connector source code? This would also imply a way for them to specify
that one of the parameters can be a "data" parameter open for FireFly to use in passing
arbitrary data. While this would allow for maximum flexibility, it's not possible within
the framework described here (mainly due to the need for a DTO mapping function that
dynamically determines if a particular method variant is able to handle a particular DTO).
