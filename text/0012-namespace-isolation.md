- Feature Name: namespace_isolation
- Start Date: 2022-05-02
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

Namespaces are a grouping construct for FireFly definitions, messages, token pools, and other objects.
They allow applications to segregate different buckets of data and functionality. However, these namespaces
are not fully isolated. They currently must share a set of plugins (a single database, blockchain, etc),
and they implicitly depend on a special "ff_system" namespace where global state is shared with network members.

To support more flexible use cases, namespaces will be extended with two additional options:
- a list of plugins (blockchain, database, data exchange, shared storage, and tokens) per namespace
- an option to control if FireFly definitions are implicitly broadcast to other namespace members

In addition, the special "ff_system" namespace will be removed.

# Motivation
[motivation]: #motivation

A FireFly supernode is designed to be an organization's web3 gateway to all the blockchain ecosystems that
they participate in - multiple blockchains, multiple token economies, multiple business networks.

Some blockchain ecosystems will be a multi-party system. This implies a decentralized application running across
a set of participants, where every member runs their own FireFly node consistently, with a copy of the same
application stack on top. FireFly is used in this case to establish and share identity, data definitions, and
data (private and broadcast) across the multiple parties.

Others will be existing blockchain ecosystems, where use of Hyperledger FireFly is optional. FireFly may be used
to simplify the processes of invoking smart contracts, interacting with digital assets, and exchanging value in
these ecosystems, but the fact that any party is utilizing FireFly can be invisible to the other participants.

This FIR represents a step forward in enabling both types of interaction, by leveraging namespaces as a way to
indepently configure:
* a set of plugins and the infrastructure components underneath (which blockchain node/IPFS gateway to use, etc)
* whether there is a multi-party network associated with the namespace, and if definitions (datatypes, locations
  of on-chain smart contracts, etc) should be implicitly shared with other members
* what on-chain data should be indexed (tokens, custom contract events, etc)
* what off-chain API security is needed to access the off-chain data store (OAuth scopes, etc)
* whether the "address book" of identities should be published to others or kept private

These changes will allow namespaces to be both more flexible and more clearly isolated from one another.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Namespaces are a construct for segregating data and operations from FireFly's perspective. They are defined
in the core config file, and fall into two basic categories:

**Multi-party namespace**

This namespace is shared with one or more other FireFly nodes. It requires three types of communication
plugins - blockchain, data exchange, and shared storage. Org and node identities must be claimed with
an identity broadcast when joining the namespace, which establishes credentials for blockchain and
data exchange communication. Shared objects can be defined in the namespace (such as datatypes and token
pools), and details of them will be implicitly broadcast to other members.

This type of namespace is used when multiple parties need to share on- and off-chain data and agree upon
the ordering and authenticity of that data.

Prior to this FIR, it was assumed that all namespaces were part of a single multi-party system of this type.

**Gateway namespace**

Nothing in this namespace will be shared automatically, and no assumptions are made about whether other
parties connected through this namespace are also using Hyperledger FireFly. Plugins for data exchange and
shared storage are currently not supported. If any identities or definitions are created in this namespace,
they will be stored in the local database, but will not be shared implicitly outside the node.

This type of namespace is mainly used when interacting directly with a blockchain, without assuming that the
interaction needs to conform to FireFly's multi-party system model.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Configuration

Plugins and namespaces are configured as follows:

```
plugins:
  database:
  - name: sqlite3
    type: sqlite3
    sqlite3:
      migrations:
        auto: true
      url: /etc/firefly/db?_busy_timeout=5000
  blockchain:
  - name: ethereum
    type: ethereum
    ethereum:
      ethconnect:
        url: http://ethconnect_0:8080
        topic: "0"
      fireflyContract:
        - address: 0x4ae50189462b0e5d52285f59929d037f790771a6
          fromBlock: 0
          toBlock: 5000
        - address: 0x3c1bef20a7858f5c2f78bda60796758d7cafff27
  dataexchange:
  - name: ffdx
    type: ffdx
    ffdx:
      url: http://dataexchange_0:3000
  sharedstorage:
  - name: ipfs
    type: ipfs
    ipfs:
      api:
        url: http://ipfs_0:5001
      gateway:
        url: http://ipfs_0:8080
  tokens:
  - name: erc20_erc721
    type: fftokens
    fftokens:
      url: http://tokens_0_0:3000

namespaces:
  default: default
  predefined:
  - name: default
    remoteName: default
    description: Default predefined namespace
    mode: multiparty
    org:
      key: 0x123456
    plugins:
    - sqlite3
    - ethereum
    - ffdx
    - ipfs
    - erc20_erc721
```

**Plugin Config**

The top-level keys for `database`, `blockchain`, `dataexchange`, `sharedstorage`, and
`tokens` will be deprecated in favor of the new ones above (nested under `plugins`).
The old keys will still be parsed if the new ones are unset (ie `plugins.blockchain` will take
precedence, but `blockchain` will be read as a fallback).

All of the new keys will now support an array of plugins (similar to how the current
`tokens` key is structured). Type-specific config will always be nested in a sub-key (meaning
the current tokens config such as `url` moves into an `fftokens` sub-key, to mirror how
other plugins are structured).

Config restrictions:
* all plugin names must be fully unique on this node (any duplicate name is a config error)

**Blockchain Config**

The `ethereum` and `fabric` plugins both have a new child key for `fireflyContract`. This is
an array of objects with an `address` or `chaincode`, and optionally `fromBlock` and `toBlock`
numbers. If a `toBlock` is specified, events past that block will be ignored, and all event
stream listeners will be moved to the next contract in the array. This enables migration from
one version of the FireFly contract to a new one.

The old `ethconnect.instance`, `ethconnect.fromBlock`, and `fabconnect.chaincode` keys are
deprecated in favor of the new ones detailed above.

**Namespace Config**

The `namespaces.predefined` objects will get these new sub-keys:
* `remoteName` is the namespace name to be sent in plugin calls, if it differs from the
  locally used name (useful for interacting with multiple shared namespaces of the same name -
  defaults to the value of `name`)
* `mode` is an enum with values `multiparty` or `gateway` (defaults to `multiparty`)
* `org.key` is a string allowing you to override the default signing key within this namespace
* `plugins` is an array of plugin names to be activated for this namespace (defaults to
  all available plugins if omitted)

Config restrictions:
* `name` must be unique on this node
* for historical reasons, "ff_system" is a reserved string and cannot be used as a `name` or `remoteName`
* a `database` plugin is required for every namespace
* if `mode: multiparty` is specified, plugins _must_ include one each of `blockchain`,
  `dataexchange`, and `sharedstorage`
* if `mode: gateway` is speicified, plugins _must not_ include `dataexchange` or `sharedstorage`
* at most one of each type of plugin is allowed per namespace, except for tokens (which
  may have many per namespace)

This should allow for graceful deprecation and sensible defaults when parsing old config files,
while enabling all of the new configuration needed in this FIR.

It will no longer be necessary to store namespaces in the database. They will be read directly
from the config into memory. For migration purposes, any existing namespaces in the database
will be read and merged into the config (with the config taking precedence).

## New FireFly Contract

This change introduces a new version of the FireFly contract for both Ethereum and Fabric. The new
contract is specific to a single namespace, so it will _not_ take a `namespace` parameter to
`pinBatch()` or emit it in the `BatchPin` event.

In addition, it contains new functions `networkVersion()` and `setNetworkVersion()`. These allow
multi-party networks to agree on the set of rules used for a given namespace. All prior versions of
the contract (without these methods) are assumed to be "network version 1". This FIR introduces
"network version 2", which primarily differs in how identities are resolved (see "Identities"
section below).

## Namespace APIs

Namespaces will now be defined only via config, so the POST API for defining new namespaces will
be removed.

## Messaging

For namespaces with the new `mode: gateway` config, all messaging (broadcast and private) will
be disabled. Determining a more direct way to interact with data exchange and shared storage
without the assumptions made by FireFly's current messaging flows is outside the scope of this FIR.

## Identities

Org and node identities must now be broadcast on a normal namespace, instead of on the special
"ff_system" namespace. The "ff_system" namespace will no longer be used. An identity must be
defined on every namespace where it is to be used, and identities can only be resolved within
a given namespace (this includes resolving DIDs).

Because this is a breaking change to identity resolution, existing multi-party networks will need
to migrate to the new identity rules all at once. This will be handled by deploying a new instance
of the FireFly contract that specifies "network version 2", and editing all member nodes' config
files to specify the block number at which the nodes will migrate from the old contract to the
new one (see preceding sections on config and contract changes).

The following top-level APIs are deprecated and replaced:
```
/network/identities - replaced by existing /namespaces/{ns}/identities
/network/identities/{did} - replaced by new /namespaces/{ns}/identities/{did}
```

The following APIs are moved from the top-level to reside under `/namespaces/{ns}`:

```
/network/diddocs/{did}
/network/nodes
/network/nodes/{nameOrId}
/network/nodes/self
/network/organizations
/network/organizations/{nameOrId}
/network/organizations/self
/status
```

For networks operating in "network version 1", FireFly will fall back to resolve identities
against the legacy "ff_system" namespace (if they are not found in the current namespace).
For networks operating in "network version 2", all identities must have been broadcast on
a namespace before using them within that namespace.

Because DIDs must now be resolved within a namespace, the formatting of custom identities as
`did:firefly:ns/{namespace}/{name}` is redundant, Custom identities now have a simpler DID
of the form `did:firefly:{name}` (but legacy DIDs including the namespace will still validate).
The character set for `name` is expanded to include `/`, so that users can define custom
identities with a DID like `did:firefly:division/acme`.

## Local Definitions

The following are all "definition" types in FireFly:
* namespace (to be removed)
* datatype
* group
* token pool
* contract interface
* contract API
* organization (deprecated)
* node (deprecated)
* identity claim
* identity verification
* identity update

For namespaces with the new `mode: gateway` config, the APIs which create these
definitions will become an immediate local database insert, instead of performing a
broadcast. Identities in this mode will not undergo any claim/verification process,
but will be created and stored locally.

## Plugin Routing (Outbound)

Nearly every manager call will now have a pre-flight namespace lookup to determine the
plugin that will handle it, instead of being able to rely on statically configured
plugins. If a namespace does not have active plugins of the type(s) needed for a
particular request, it will reject with HTTP 400.

If the namespace has a `remoteName` configured, it will be substituted in before calling
out to the plugin.

## Event Routing (Inbound)

Each namespace+plugin combo will represent a unique event listener loop (might mean
many copies of the same loop, if a plugin is shared across multiple namespaces).

If the namespace has a `remoteName` configured, the event loop will substitute in the
local `name` before processing the event.

## FireFly CLI

The CLI will be enhanced to support defining a default namespace with `mode: gateway`
(which also implies no org/node registration). The default behavior of the CLI will continue
to be generating a "multiparty" namespace.

# Drawbacks
[drawbacks]: #drawbacks

This is a fairly dramatic expansion of what a "namespace" means in FireFly, so the
migration (both in technical terms and in terms of how users should think about it)
is complex. It should be possible to migrate environments without breaking existing use
cases, but there is definitely some potential for edge cases we haven't considered.

By defining all namespaces in the config file, the likelihood of namespaces changes
between application starts is much higher. Names, plugins, and configuration details
may be liable to change at any point, giving users more opportunities to create
problems via invalid config.

Removing the ability to dynamically broadcast new namespaces may introduce a burden if
any users rely on this functionality to define namespaces without editing the
config on every node. Adding back this requirement in the new landscape would require
some notion of "child" namespaces that can be created via a
broadcast on an existing multi-party namespace (not currently in scope for
this FIR).

# Rationale and alternatives
[alternatives]: #alternatives

Rather than redefining "namespaces", a new construct could be introduced to serve these
additional use cases. While this would mitigate the migration impact, it would overall
represent a larger code change to add the new construct to all APIs across the system.
Since namespaces are already pervasive but not 100% isolated, reworking them with clearer
isolation seems like a reasonable route.

# Prior art
[prior-art]: #prior-art

There are some similarities to Kubernetes namespaces:
https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces

or to PostgreSQL tablespaces:
https://www.postgresql.org/docs/current/manage-ag-tablespaces.html

# Testing
[testing]: #testing

Migration testing of existing environments of various configs will be critical to avoid
breaking things as a result of this change.

New E2E tests for nodes with different combinations of namespace configs will also need to be added.

# Dependencies
[dependencies]: #dependencies

[#792](https://github.com/hyperledger/firefly/pull/792) may ease some of the migration, because many
relocated URLs (like `/status` and `/network/organizations`) will continue working by simply
switching to query the default namespace when a namespace is omitted from the URL path.

# Unresolved questions
[unresolved]: #unresolved-questions

* By moving the `/status` route to be namespaced, but not child routes such as `/status/websockets`,
  it creates a confusing division of APIs. May need further discussion and renaming of these routes.
* Currently the names configured for tokens plugins must agree across all members of a consortium. It
  would be nice to remove this restriction as part of this change.
