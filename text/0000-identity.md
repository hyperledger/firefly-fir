---
layout: default
title: Extensible Identity Resolution
nav_order: 3
---

- Feature Name: extensible_identity_resolution
- Start Date: 2022-01-07
- FIR PR: (leave this empty)
- FireFly Component: Identity Manager
- FireFly Issue: (leave this empty)

# Summary

[summary]: #summary

Provide a global scheme for specifying signing identities, which can be resolved to the associated signing keys to use with the different underlying DLTs that FireFly uses.

# Motivation

[motivation]: #motivation

As of now, FireFly defines a `key` construct for objects that needs signing keys, using the scheme taken directly from the underlying DLT protocols. For instance, when used with Ethereum, a `key` value takes the `0x1234...abcd` format, whereas with Fabric, a `key` value can be either the short name like `user1`, or the fully resolved string `org1::x509::CN=fabric-ca::CN=user1`.

As a result, the DLT specific values can be spreading across an application code base using FireFly. This is not optimal as FireFly strives to mask differences among the different DLT technologies as much as possible.

In addition, FireFly also defines an `author` construct that identifies the signer, but still requires the separate `key` property to be provided alongside. Where we'd like for FireFly to be is for the signing keys to bound under the signer's identifier, so that the signer identifier is the only required parameter. This is also critical to supporting key rotations. Currently when a signing key is rotated, the application configuration must be updated accordingly which is not ideal.

To find a global identity scheme that supports resolution to (possibly multiple) signing keys, and can support any DLT protocols, we need not look any further than the DID technologies (Decentralized Identifiers). This is a W3C specification that defines a universal mechanism to represent an identity (people, orgs, things) that holds cryptographic keys. The design of the DID architecture consists of two high level parts: the `DID string` and the `DID document`. The string is used as the unique identitier that resolves to the document which provides rich information about the identity, such as public keys and verification methods.

Details of the DID specification can be found at [https://www.w3.org/TR/did-core/](https://www.w3.org/TR/did-core/).

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

Given the already defined namespace `did:firefly`, we want to complete the DID support by making the [identity manager](https://github.com/hyperledger/firefly#firefly-core-code-hierarchy) return a [DID Document](https://www.w3.org/TR/did-core/#example-1-a-simple-did-document).

Given an organization identity string `did:firefly:org/acme`, a DID Document like the following must be returned by the identity manager:

```
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ]
  "id": "did:firefly:org/acme",
  "verificationMethod": [{
    "id": "did:firefly:org/acme#0xb9c5714089478a327f09197987f16f9e5d936e8a",
    "type": "EcdsaSecp256k1VerificationKey2019",
    "controller": "did:firefly:org/acme",
    "blockchainAcountId": "0xb9c5714089478a327f09197987f16f9e5d936e8a"
  }, {
    "id": "did:firefly:org/acme#keys-2",
    "type": "X25519KeyAgreementKey2019",
    "controller": "did:firefly:org/acme",
    "publicKeyMultibase": "z9hFgmPVfmBZwRvFEyniQDBkz9LmV7gDEqytWyGZLmDXE"
  }],
  "authentication": [
    "#0xb9c5714089478a327f09197987f16f9e5d936e8a"
  ],
  "keyAgreement": [
    "#keys-2"
  ]
}
```

In the above example, the first verification method is based on an secp256k1 key used for an Ethereum based blockchain. The public key is not specified but instead an `blockchainAccountId` is specified with the standard 20-byte account address used by the Ethereum community in the place of the public key.

Below is an example of a Fabric based verification method entry:

```
{
  "id": "did:firefly:org/acme#user1",
  "alsoKnownAs": "did:firefly:org/acme#mspIdForAcme::x509::CN=fabric-ca::CN=user1",
  "controller": "did:firefly:org/acme",
  "publicKey": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9...wQIDAQAB\n-----END PUBLIC KEY-----"
}
```

The `type` property is omitted for now because the values for popular curves used by the Fabric community, secp256r1 etc., have not been defined (https://w3c-ccg.github.io/security-vocab), but also because the certificate already contains algorithm information on the signing key.

The `alsoKnownAs` property specifies the binding between the short form (`user1`) and the long form (`mspIdForAcme::x509::CN=fabric-ca::CN=user1`) of the Fabric signing identity ID. A reciprocal DID document must exist to provide the reverse binding:

```
{
  "id": "did:firefly:org/acme#mspIdForAcme::x509::CN=fabric-ca::CN=user1",
  "alsoKnownAs": "did:firefly:org/acme#user1",
  "controller": "did:firefly:org/acme",
  "publicKey": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9...wQIDAQAB\n-----END PUBLIC KEY-----"
}
```

The `authentication` section specifies the public key to use when authenticating an incoming signed request.

The `keyAgreement` section specifies the public key to use when negotiating encryption keys for secure data exchange.

Other [Verification Relationships](https://www.w3.org/TR/did-core/#verification-relationships) are currently not used in the context of FireFly.

With the DID Document resolution given the DID string, we should be able to remove the `key` property from the payload and require only the `author` field. The reference to the signing key can be obtained by parsing the DID document and locating the key used by the `authentication` specification of the document. Using the reference to the public key, the private key can then be located in the wallet managed by the FireFly blockchain plugin.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

No additional information is needed here.

# Drawbacks

[drawbacks]: #drawbacks

# Rationale and alternatives

[alternatives]: #alternatives

There are existing implementations of DIDs for Ethereum, and Fabric, such as [the Ethereum DID Resolver](https://github.com/decentralized-identity/ethr-did-resolver) and [Hyperledger Labs TrustID](https://github.com/hyperledger-labs/trustid).

Both of the above, and most of the other existing implementations, are based on the underlying DLT ledger (or blockchain) to maintain the DID registry and resolve the DID documents. FireFly already provides a pluggable mechansim to register and broadcast identities. Registries based on blockchains is but one type of plugin.

[trustbloc/orb](https://github.com/trustbloc/orb) is an implementation of the [DIF SideTree](https://identity.foundation/sidetree/spec/) specification, which describes a network of nodes that manages the lifecycle (CRUD) operations and propagation of DIDs. The spec focuses on a trustless design that requires witnesses to provide proofs for a proposed identity, and relies on an underlying DLT/blockchain for a globally ordered history of operations on the DID. FireFly strives to provide a flexible suite of implementations for managing identities within the multiparty system. This goal implies FireFly must support identity registries that have different trust assumptions. In addition to a trustless design as in SideTree, a centralized approach can be appropriate for many use cases.

# Prior art

[prior-art]: #prior-art

The projects mentioned above made up the bulk of the prior art relevant to this proposal.

# Testing

[testing]: #testing

Existing unit tests and e2e tests will be sufficient to test the new feature, after being adapted to work with the new approach.

# Dependencies

[dependencies]: #dependencies

None

# Unresolved questions

[unresolved]: #unresolved-questions

- what other schemes than `did:firefly:org` should be considered, for FireFly to support more types of identities than organizations? `did:firefly:node`, `did:firefly:user`?
- the Data Exchange component should switch to using the DID Document based key agreement set up, but it's not clear if there are practical benefits than architectural consistency. More discussions with the developers of that components are needed
