- Feature Name: namespaced_plugins
- Start Date: 2022-05-02
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

Namespaces are a grouping construct for FireFly definitions, messages, token pools, and other objects.
They allow applications to segregate different buckets of data and functionality by filtering on a
namespace. However, data from all namespaces still share the same blockchain, database, and other plugins.

To support more flexible use cases, namespaces will be extended with two additional per-namespace options:
- a list of plugins (blockchain, database, data exchange, shared storage, tokens)
- an option for FireFly definitions to be broadcast (default) or local-only

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

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Namespaces are a construct for segregating data and operations from FireFly's perspective. They are defined
in the core config file, and fall into two basic categories:

**Broadcast Namespace**

This namespace is shared with one or more other FireFly nodes. It requires three types of communication
plugins (blockchain, data exchange, and shared storage). Org and node identities must be claimed with
an identity broadcast when joining the namespace, which establishes credentials for blockchain and
data exchange communication. Shared objects can be defined in the namespace (such as datatypes and token
pools), and details will be broadcast to other members.

This type of namespace is used when multiple parties need to share on- and off-chain data and agree upon
the ordering and authenticity of that data.

Prior to this FIR, this was the only type of namespace.

**Local Namespace**

This namespace is defined for the local node only. There are no restrictions on which plugins are required
(although the active plugins will determine which FireFly functionality can be exercised within the namespace).
If any identities or definitions are created in this namespace, they will be immediately stored in the local
database, and will not be shared outside the node.

This type of namespace is mainly used when interacting directly with a blockchain, without the aggregation
and verification needs of a multi-party broadcast namespace.

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
    alias: default
    description: Default predefined namespace
    definitions: broadcast
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
* `alias` is a local name for the namespace (useful for interacting with multiple shared
  namespaces of the same name - defaults to `name` if omitted)
* `definitions` is an enum with values `broadcast` (the default) or `local`
* `plugins` is an array of plugin names to be activated for this namespace (defaults to
  all available plugins if omitted)

Config restrictions:
* `alias` must be unique on this node
* for historical reasons, "ff_system" is a reserved string and cannot be used as an `alias`
* if two namespaces share the same remote `name`, they cannot share any plugins in common
* a `database` plugin is required for every namespace
* if `definitions: broadcast` is specified, plugins must include one each of `blockchain`,
  `dataexchange`, and `sharedstorage`
* at most one of each type of plugin is allowed per namespace, except for tokens (which
  may have many per namespace)

This should allow for graceful deprecation and sensible defaults when parsing old config files,
while enabling all of the new configuration needed in this FIR.

It will no longer be necessary to store namespaces in the database. They will be read directly
from the config into memory. For migration purposes, any existing namespaces in the database
will be read and merged into the config (with the config taking precedence).

TBD: is the "default" namespace still required, and does it have any restrictions?

## Namespace APIs

Namespaces will now be defined only via config, so the POST API for defining new namespaces will
be removed.

## Identities

Org and node identities must now be broadcast on a normal namespace, instead of on the special
"ff_system" namespace. The "ff_system" namespace will no longer be used. Top-level org and node
APIs must be deprecated or removed, and new ones added under `/namespaces/{ns}`.

When resolving an identity, FireFly will first check against the current namespace, and then
(for historical reasons) against the "ff_system" namespace.

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

For namespaces with the new `definitions: local` config, the APIs which create these
definitions will become an immediate local database insert, instead of performing a
broadcast. This may mean restructuring the Definition Handler to be callable from
other managers (as currently it is only invoked by the Event Manager).

## Plugin Routing (Outbound)

Nearly every manager call will now have a pre-flight namespace lookup to determine the
plugin that will handle it, instead of being able to rely on statically configured
plugins. If a namespace does not have active plugins of the type(s) needed for a
particular request, it will reject with HTTP 400.

If the namespace has a local `alias`, it will be replaced with the remote namespace
`name` before calling out to the plugin.

## Event Routing (Inbound)

When processing events, each event will be parsed against the configured namespaces to
find a match on two dimensions:
* event `namespace` must match the namespace `name`
* source plugin for the event must match an active plugin on the namespace

If no matching namespace is found, the event will be dropped. If the matching namespace
also has an `alias`, the `alias` will be substituted before persisting any database
objects.

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
