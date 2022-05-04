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

Currently FireFly nodes function as part of exactly one consortium. They require three shared communication
pipes with the other members (blockchain, data exchange, shared storage), and all definitions (identities,
datatypes, token pools, etc) are broadcast to other members by default. This satisfies one very common
enterprise use case, but is not flexible enough.

This change will enable two important points of flexibility:
- a FireFly node can communicate with many different consortia, using different communication pipes for each one
- a FireFly node can function as an ad-hoc gateway to any blockchain, without requiring things like FireFly
  identities and definitions to be broadcast on that chain

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
plugins (blockchain, data exchange, and shared storage). Org and node identities must be claimed with
an identity broadcast when joining the namespace, which establishes credentials for blockchain and
data exchange communication. Shared objects can be defined in the namespace (such as datatypes and token
pools), and details of them will be implicitly broadcast to other members.

This type of namespace is used when multiple parties need to share on- and off-chain data and agree upon
the ordering and authenticity of that data.

Prior to this FIR, it was assumed that all namespaces were part of a single multi-party system of this type.

**Gateway namespace**

Nothing in this namespace will be shared automatically, and no assumptions are made about whether other
parties connected through this namespace are also using Hyperledger FireFly. There are no restrictions on which
plugins are required (although the active plugins will determine which FireFly functionality can be exercised
within the namespace). If any identities or definitions are created in this namespace, they will be stored in
the local database, but will not be shared implicitly outside the node.

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
        instance: 0x4ae50189462b0e5d52285f59929d037f790771a6
        topic: "0"
        url: http://ethconnect_0:8080
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
    networkMode: shared
    identityFallbackEnabled: true
    plugins:
    - sqlite3
    - ethereum
    - ffdx
    - ipfs
    - erc20_erc721
```

The top-level keys for `database`, `blockchain`, `dataexchange`, `sharedstorage`, and
`tokens` will be deprecated in favor of the new ones above (nested under `plugins`).
All of these new keys will now support an array of plugins (similar to how the current
`tokens` key is structured). Type-specific config will always be nested in a sub-key (meaning
the current tokens config such as `url` moves into an `fftokens` sub-key, to mirror how
other plugins are structured). Config restrictions:
* all plugin names must be fully unique on this node (any duplicate name is a config error)

The `namespaces.predefined` objects will get three new sub-keys:
* `remoteName` is the namespace name to be recorded in plugin calls, if it differs from the
  locally used name (useful for interacting with multiple shared namespaces of the same name -
  defaults to the value of `name`)
* `networkMode` is an enum with values `shared` or `local` (defaults to `shared`)
* `identityFallbackEnabled` is a flag to allow resolving legacy identities defined on "ff_system"
  (defaults to `true`)
* `plugins` is an array of plugin names to be activated for this namespace (defaults to
  all available plugins if omitted)

Config restrictions:
* `name` must be unique on this node
* for historical reasons, "ff_system" is a reserved string and cannot be used as a `name` or `remoteName`
* a `database` plugin is required for every namespace
* if `networkMode: shared` is specified, plugins must include one each of `blockchain`,
  `dataexchange`, and `sharedstorage`
* at most one of each type of plugin is allowed per namespace, except for tokens (which
  may have many per namespace)

This should allow for graceful deprecation and sensible defaults when parsing old config files,
while enabling all of the new configuration needed in this FIR.

It will no longer be necessary to store namespaces in the database. They will be read directly
from the config into memory. For migration purposes, any existing namespaces in the database
will be read and merged into the config (with the config taking precedence).

TBD: is the "default" namespace still required, and does it have any restrictions?

TBD: do plugin names have to agree across all members of a consortium?

## Namespace APIs

Namespaces will now be defined only via config, so the POST API for defining new namespaces will
be removed.

## Identities

Org and node identities must now be broadcast on a normal namespace, instead of on the special
"ff_system" namespace. The "ff_system" namespace will no longer be used. Top-level org and node
APIs must be deprecated or removed, and new ones added under `/namespaces/{ns}`.

If the `identityFallbackEnabled` flag is enabled for a namespace when resolving an identity,
FireFly will first check against that namespace, and then against the legacy "ff_system"
namespace.

TBD: How to adjust org/node details reported by the `/status` API?

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

For namespaces with the new `networkMode: local` config, the APIs which create these
definitions will become an immediate local database insert, instead of performing a
broadcast. This means restructuring the definition methods (both event senders and
event processors) to be more easily callable from other managers.

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

Removing the ability to broadcast new namespaces on the fly may introduce a burden if
any users rely on this functionality to define lots of namespaces without editing the
config on every node. Satisfying this requirement in the new landscape would require
introducing some notion of "child" namespaces that can be dynamically created via a
broadcast on an existing multi-party namespace, which is not currently in scope for
this FIR.

# Rationale and alternatives
[alternatives]: #alternatives

Rather than redefining "namespaces", a new construct could be introduced to serve these
additional use cases. While this would mitigate the migration impact, it would overall
represent a larger code change to add the new construct to all APIs across the system.
Since namespaces are already pervasive but not 100% isolated, reworking them with clearer
isolation seems like a reasonable route.

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

No known dependencies

# Unresolved questions
[unresolved]: #unresolved-questions

TODO
