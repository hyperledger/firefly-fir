- Feature Name: namespace_isolation
- Start Date: 2022-05-02
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

Namespaces are a grouping construct for most of the objects within FireFly (messages, data, tokens, etc).
They allow applications to segregate different buckets of data and functionality.
However, there are a few global items shared between namespaces - such as a single database and a single
blockchain - as well as a special "ff_system" namespace used to broadcast pieces of global state.

To support more flexible use cases, namespaces will be extended so that every piece of FireFly's
configuration can be different per namespace. Specifically, this means each namespace can be configured
with a different set of plugins, all previously "global" operations will instead happen within the context
of a namespace, and the special "ff_system" namespace will be removed.

In addition, "multi-party" mode will become an optional feature for each namespace. When multi-party
mode is not enabled, a namespace can operate as a simple gateway for interacting with a blockchain. When
multi-party mode is enabled, the namespace is assumed to be shared with one or more other FireFly nodes,
and can leverage all of FireFly's capabilities for communicating and syncing state via on- and off-chain
messaging.

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
independently configure:
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

**Gateway namespace**

Nothing in this namespace will be shared automatically, and no assumptions are made about whether other
parties connected through this namespace are also using Hyperledger FireFly. Plugins for data exchange and
shared storage are currently not supported. If any identities or definitions are created in this namespace,
they will be stored in the local database, but will not be shared implicitly outside the node.

This type of namespace is mainly used when interacting directly with a blockchain, without assuming that the
interaction needs to conform to FireFly's multi-party system model.

**Multi-party namespace**

This namespace is shared with one or more other FireFly nodes. It requires three types of communication
plugins - blockchain, data exchange, and shared storage. Org and node identities must be claimed with
an identity broadcast when joining the namespace, which establishes credentials for blockchain and
data exchange communication. Shared objects can be defined in the namespace (such as datatypes and token
pools), and details of them will be implicitly broadcast to other members.

This type of namespace is used when multiple parties need to share on- and off-chain data and agree upon
the ordering and authenticity of that data.

Prior to this FIR, it was assumed that all namespaces were part of a single multi-party system of this type.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Configuration

Plugins and namespaces are configured as follows:

```
plugins:
  database:
  - name: database0
    type: sqlite3
    sqlite3:
      migrations:
        auto: true
      url: /etc/firefly/db?_busy_timeout=5000
  blockchain:
  - name: blockchain0
    type: ethereum
    ethereum:
      ethconnect:
        url: http://ethconnect_0:8080
        topic: "0"
  dataexchange:
  - name: dataexchange0
    type: ffdx
    ffdx:
      url: http://dataexchange_0:3000
  sharedstorage:
  - name: sharedstorage0
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
    defaultKey: 0x123456
    plugins: [database0, blockchain0, dataexchange0, sharedstorage0, erc20_erc721]
    multiparty:
      enabled: true
      org:
        name: org0
        description: org0
        key: 0x123456
      contract:
        - location:
            address: 0x4ae50189462b0e5d52285f59929d037f790771a6
          fromBlock: 0
        - location:
            address: 0x3c1bef20a7858f5c2f78bda60796758d7cafff27
          fromBlock: 5000
```

**Plugin Config**

The top-level keys for `database`, `blockchain`, `dataexchange`, `sharedstorage`, and
`tokens` will be deprecated in favor of the new ones above (nested under `plugins`).
The old keys will still be parsed if the new ones are unset (ie `plugins.blockchain` will take
precedence, but `blockchain` will be read as a fallback).

All of the new keys will now support an array of plugins. Each array entry will have a `name`
and `type`, and sub-keys for each possible type (so database plugins have sub-keys for
`sqlite3` and `postgres`, tokens plugins only have a sub-key for `fftokens`, etc).

Config restrictions:
* all plugin names must be fully unique on this node (any duplicate name is a config error)

**Blockchain Config**

A new `/network/action` API is added, with a single supported action expressed as
`{"type": "terminate"}`. This action will send a special blockchain transaction that signals all
network members to unsubscribe from their current contract and move to the next one configured in
the contract list.

The configuration options that referece FireFly's multiparty contract (`ethconnect.instance`,
`ethconnect.fromBlock`, and `fabconnect.chaincode`) are deprecated in favor of new multiparty-specific
keys detailed in the next section (but the old config will still be read as a fallback).

**Namespace Config**

The `namespaces.predefined` objects will get these new sub-keys:
* `remoteName` is the namespace name to be sent in plugin calls, if it differs from the
  locally used name (useful for interacting with multiple shared namespaces of the same name -
  defaults to the value of `name`)
* `defaultKey` is a blockchain key used to sign transactions when none is specified (in multiparty mode,
  defaults to the org key)
* `plugins` is an array of plugin names to be activated for this namespace (defaults to
  all available plugins if omitted)
* `multiparty.enabled` controls if multi-party mode is enabled (defaults to true if an org key or
  org name is defined on this namespace _or_ in the deprecated `org` section at the root)
* `multiparty.org` is the root org identity for this multi-party namespace (containing `name`,
  `description`, and `key`)
* `multiparty.contract` is an array of objects describing the location(s) of a FireFly multiparty
  smart contract. Its children are blockchain-agnostic `location` and `firstEvent` fields, with formats
  identical to the same fields on custom contract interfaces and contract listeners. The blockchain plugin
  will interact with the first contract in the list until instructions are received to terminate it and
  migrate to the next.

Config restrictions:
* `name` must be unique on this node
* for historical reasons, "ff_system" is a reserved string and cannot be used as a `name` or `remoteName`
* a `database` plugin is required for every namespace
* if `multiparty.enabled` is true, plugins _must_ include one each of `blockchain`, `dataexchange`, and
  `sharedstorage`
* if `multiparty.enabled` is false, plugins _must not_ include `dataexchange` or `sharedstorage`
* at most one of each type of plugin is allowed per namespace, except for tokens (which
  may have many per namespace)

This should allow for graceful deprecation and sensible defaults when parsing old config files,
while also enabling all of the new configuration needed in this FIR.

It will no longer be necessary to store namespaces in the database. They will be read directly
from the config into memory. For migration purposes, any existing namespaces in the database
will be read and merged into the config (with the config taking precedence).

**Org Config**

The `org` section at the root is deprecated in favor of the per-namespace `multiparty.org` section
detailed above (but the old config will be honored for every multiparty namespace as a fallback).

## New FireFly Contract

This change introduces a new version of the FireFly contract for both Ethereum and Fabric. The new
contract is specific to a single namespace, so it will _not_ take a `namespace` parameter to
`pinBatch()` or emit it in the `BatchPin` event.

In addition, it contains a new function, `networkVersion()`. This allows
multi-party networks to agree on the set of rules used for a given namespace. All prior versions of
the contract (without these methods) are assumed to be "network version 1". This FIR introduces
"network version 2", which differs in the signature of BatchPin as detailed above, and in how
identities are resolved (see "Identities" section below).

## Admin Config API

Because it represents global state and was not widely adopted, the admin API for modifying config
values via HTTP will be removed. Config will only be read from the config file.

## Namespace APIs

Namespaces will now be defined only via config, so the POST API for defining new namespaces will
be removed.

## Messaging

For gateway namespaces (ie `multiparty.enabled = false`), all messaging (broadcast and private) will
be disabled. Determining a more direct way to interact with data exchange and shared storage
without the assumptions made by FireFly's current messaging flows is outside the scope of this FIR.

## Identities

Org and node identities must now be broadcast on a normal namespace, instead of on the special
"ff_system" namespace. The "ff_system" namespace will no longer be used. An identity must be
defined on every namespace where it is to be used, and identities can only be resolved within
a given namespace (this includes resolving DIDs).

Because this is a breaking change to identity resolution, existing multi-party networks will need
to migrate to the new identity rules together. This will be handled by deploying a new instance
of the FireFly contract that specifies "network version 2", and editing all member nodes' config
files to contain an entry for the new contract. One member will then use the `/network/action`
API to broadcast a `terminate` action, which will cause all members to stop listening to the old
contract, move to the new one, and therefore begin following the rules of "network version 2"
(including ignoring the "ff_system" namespace).

The following top-level APIs are deprecated and replaced:
```
/network/identities - replaced by existing /namespaces/{ns}/identities
/network/identities/{did} - replaced by new /namespaces/{ns}/identities/{did}
```

The following top-level APIs are moved (due to perceived lower usage of these routes,
the old paths are removed instead of being deprecated):
```
/status/pins - moved to /namespaces/{ns}/pins
/status/websockets - moved to /namespaces/{ns}/websockets
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
/status/batchmanager
```

For networks operating in "network version 1", FireFly will fall back to resolve identities
against the legacy "ff_system" namespace (if they are not found in the current namespace).
For networks operating in "network version 2", all identities must have been broadcast on
a namespace before using them within that namespace.

Because DIDs must now be resolved within a namespace, the formatting of custom identities as
`did:firefly:ns/{namespace}/{name}` is redundant, Custom identities now have a simpler DID
of the form `did:firefly:{name}` (but legacy DIDs including the namespace will still validate).

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

For gateway namespaces, the APIs which create these definitions will become an immediate
local database insert, instead of performing a broadcast. Additional caveats:
* identities in this mode will not undergo any claim/verification process,
  but will be created and stored locally
* datatypes and groups will not be supported, as they are only useful in the context
  of messaging (which is disabled in gateway namespaces)

## Namespace Manager

A new component called Namespace Manager is introduced, at a level above Orchestrator.
The Namespace Manager will parse and validate the plugin and namespace config, then
create a separate Orchestrator for each namespace (along with separate instances of
every manager needed by that namespace). Plugins may be shared across many namespaces.

## Plugin Routing (Outbound)

If a namespace does not have active plugins of the type(s) needed for a
particular request, it will reject with HTTP 400.

If the namespace has a `remoteName` configured, it will be substituted in before calling
out to the plugin.

## Event Routing (Inbound)

If the namespace has a `remoteName` configured, the event manager will substitute in the
local `name` before processing the event.

## FireFly CLI

The CLI will be enhanced to support defining a default namespace without multiparty mode enabled
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

* Currently the names configured for tokens plugins must agree across all members of a consortium. It
  would be nice to remove this restriction as part of this change.
