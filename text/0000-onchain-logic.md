---
layout: default
title: Custom On-Chain Logic
nav_order: 0
---

- Feature Name: Custom On-Chain Logic
- Start Date: 2021-11-10
- FIR PR: (leave this empty)
- FireFly Component: firefly
- FireFly Issue: (leave this empty)

# Summary

[summary]: #summary

The Custom On-Chain Logic introduces a new set of unified APIs for FireFly to invoke any smart contract and subscribe to events emitted from that contract. It also introduces a consistent way to describe the interface of a smart contract, including its functions, parameters, return values, and events, in a way that is mostly abstracted from a specific blockchain implementation.

> Note: Deployment of smart contracts is not in scope of this improvement request.

# Motivation

[motivation]: #motivation

Many FireFly users have requested the ability to interact with custom smart contracts through FireFly. This is one of the major features planned for v1.0. Right now, custom smart contracts can be invoked directly through FireFly's blockchain connectors, but these APIs are specific to each blockchain connector. Today, there is no way to change blockchain implementations without also significantly changing API calls to interact with a custom contract. The goal of this feature is to have a set of APIs in the FireFly Core which allow users to swap out FireFly plugins such as the blockchain connector, without needing to change any API calls made to FireFly's API.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

With this enhancement, FireFly exposes a simple set of APIs to work with custom smart contracts. Given a smart contract on a blockchain, these APIs will allow an end user to:

1. Directly invoke or query a smart contract while providing all required schema and location / ledger information in a single request.

   | Method | Endpoint                            | Description                                  |
   | ------ | ----------------------------------- | -------------------------------------------- |
   | `POST` | `/namespaces/{ns}/contracts/invoke` | Invoke a method on a smart contract          |
   | `POST` | `/namespaces/{ns}/contracts/query`  | Query a smart contract (read-only operation) |
   | `GET`  | `/namespaces/{ns}/contracts/query`  | Query a smart contract (read-only operation) |

1. Subscribe to events emitted from smart contracts while providing all required schema and location / ledger information in a single request. Standard CRUD style REST operations can be used on this endpoint to manage subscriptions as well. If the subscription is for an event emitted by a contract described by a Contract Interface that has already been registered, the Contract Interface may be referenced in the request to create a subscription, rather than defining the entire event schema inline.

   | Method   | Endpoint                                              | Description           |
   | -------- | ----------------------------------------------------- | --------------------- |
   | `POST`   | `/namespaces/{ns}/contracts/subscriptions`            | Create a subscription |
   | `GET`    | `/namespaces/{ns}/contracts/subscriptions`            | List subscriptions    |
   | `GET`    | `/namespaces/{ns}/contracts/subscriptions/{nameOrId}` | Get a subscription    |
   | `DELETE` | `/namespaces/{ns}/contracts/subscriptions/{nameOrId}` | Delete a subscription |

1. Describe the interface of the smart contract including its functions and events in a standardized format. This format an FFI (FireFly Interface). Contract interfaces require a name and version that is unique within their namespace. When a new contract is defined, it is also sent as a broadcast to other members of the network.

   | Method | Endpoint                                     | Description                                                                            |
   | ------ | -------------------------------------------- | -------------------------------------------------------------------------------------- |
   | `POST` | `/namespaces/{ns}/contracts/interfaces`      | Define a new smart contract interface and broadcast it to other members of the network |
   | `GET`  | `/namespaces/{ns}/contracts/interfaces`      | List contract interfaces                                                               |
   | `GET`  | `/namespaces/{ns}/contracts/interfaces/{id}` | Get a contract interface                                                               |

1. Invoke or query a smart contract that implements a given interface while providing its location / ledger information (contract address or channel / chaincode, etc.) inline in the request

   | Method | Endpoint                                                         | Description                                  |
   | ------ | ---------------------------------------------------------------- | -------------------------------------------- |
   | `POST` | `/namespaces/{ns}/contracts/interfaces/{id}/invoke/{methodPath}` | Invoke a method on a smart contract          |
   | `POST` | `/namespaces/{ns}/contracts/interfaces/{id}/query/{methodPath}`  | Query a smart contract (read-only operation) |
   | `GET`  | `/namespaces/{ns}/contracts/interfaces/{id}/query/{methodPath}`  | Query a smart contract (read-only operation) |

1. Register a set of named API endpoints that allow a user to interact with a defined smart contract interface. This will also generate an OpenAPI document that describes how to use each endpoint. Optionally, the location and ledger information (contract address or channel / chaincode, etc.) of the contract can part of the API registration request, or it can be specified inline with each invoke/query request.

   | Method | Endpoint                                           | Description                                                                          |
   | ------ | -------------------------------------------------- | ------------------------------------------------------------------------------------ |
   | `POST` | `/namespaces/{ns}/apis`                            | Create a named API                                                                   |
   | `GET`  | `/namespaces/{ns}/apis`                            | List named APIs                                                                      |
   | `GET`  | `/namespaces/{ns}/apis/{name}`                     | Get a named API                                                                      |
   | `PUT`  | `/namespaces/{ns}/apis/{name}`                     | Update a named API - can be used to update the contract interface this API points to |
   | `POST` | `/namespaces/{ns}/apis/{name}/invoke/{methodPath}` | Invoke a method on a smart contract                                                  |
   | `POST` | `/namespaces/{ns}/apis/{name}/query/{methodPath}`  | Query a smart contract (read-only operation)                                         |
   | `GET`  | `/namespaces/{ns}/apis/{name}/query/{methodPath}`  | Query a smart contract (read-only operation)                                         |

1. Query the API for events related to custom smart contracts.
   | Method | Endpoint | Description |
   | -------- | ------------------------------------------------- | ------------------------------------------------------------------------------------ |
   | `GET` | `/namespaces/{ns}/contracts/events` | List contract events |
   | `GET` | `/namespaces/{ns}/contracts/events/{id}` | Get a contract event by ID |

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Contract Interfaces

- Describe the interface of a smart contract to enable FireFly to create an API to interact with them
- Are namespaced, like most other FireFly resources
- Must have a unique name and version combination within their namespace
- When a contract interface is created in FireFly, it is broadcasted to all other members in the network, via the existing broadcast function of FireFly. It is broadcasted on the same namespace that it is created in.
- Will be stored in a new table in the FireFly Core database called `contractinterfaces` for fast access during runtime
- Each of its methods and params (or returns) are also stored in their own tables `contractmethods` and `contractparams` respectively, for fast access during runtime.

### Proposed JSON representation

```json
{
  "name": "simple-storage",
  "version": "v0.1.0",
  "methods": [
    {
      "name": "set",
      "params": [
        {
          "name": "newValue",
          "type": "integer",
          "details": {
            "type": "uint256"
          }
        }
      ],
      "returns": []
    },
    {
      "name": "get",
      "params": [],
      "returns": [
        {
          "name": "output",
          "type": "integer",
          "details": {
            "type": "uint256"
          }
        }
      ]
    }
  ],
  "events": [
    {
      "name": "Changed",
      "params": [
        {
          "name": "from",
          "type": "string",
          "details": {
            "type": "address"
          }
        },
        {
          "name": "value",
          "type": "integer",
          "details": {
            "type": "uint256"
          }
        }
      ]
    }
  ]
}
```

### Retrieving / listing contract interfaces

By default, the entire interface is not listed when the interface is fetched by the API. An optional query parameter `?interface=true` can be included to list the full interface.

```json
[
  {
    "id": "933db039-8240-4bd7-b262-1a92f13ed0a2",
    "namespace": "default",
    "name": "simple-storage",
    "version": "v0.1.0"
  }
]
```

## Contract API

- Points to a specific instance of a smart contract on-chain
- Are namespaced, like most other FireFly resources
- Is assigned a friendly name that must be unique within the namespace. This will be used as part of the API path
- When a contract API is created in FireFly, it is broadcasted to all other members in the network, via the existing broadcast function of FireFly. It is broadcasted on the same namespace that it is created in.
- Will be stored in a new table in the FireFly Core database called `contractapis` for fast access during runtime.

### Proposed JSON representation

### Creating a contract API

```json
{
  "contract": {
    "id": "933db039-8240-4bd7-b262-1a92f13ed0a2"
  },
  "location": {
    "address": "0xf6cdd9fdb855097edcfb45e1caba980f0e1517bb"
  },
  "ledger": {
    ...
  },
  "name": "simple-storage"
}
```

> **NOTE**: The `ledger` field is optionally present anywhere an on-chain location is being referenced. This field has been added here for future compatibility when multi-ledger support is fully implemented. The value of the `ledger` field is passed to the blockchain plugin as a byte array so each blockchain plugin can have a specific schema within the `ledger` field. Currently, the value is simply stored, but not used.

### Looking up a contract instance

```json
[
  {
    "id": "33bcdf7e-3fbe-4ea9-99db-4219f4034242",
    "namespace": "default",
    "contract": {
      "id": "933db039-8240-4bd7-b262-1a92f13ed0a2"
    },
    "ledger": {
      ...
    },
    "location": {
      "address": "0xf6cdd9fdb855097edcfb45e1caba980f0e1517bb"
    },
    "name": "simple-storage"
  }
]
```

### Invoking a smart contract

```json
{
  "location": {
    "address": "0xf6cdd9fdb855097edcfb45e1caba980f0e1517bb"
  },
  "method": {
    "name": "set",
    "params": [
      {
        "name": "newValue",
        "type": "int"
      }
    ],
    "returns": []
  },
  "input": {
    "newValue": 120
  }
}
```

### FireFly Contract Interface Types

The FFI format defines several straightforward types that can be used in custom smart contracts. They are as follows:

- `boolean`
- `integer`
- `string`
- `byte[]` (Base64 encoded string. Alternatively, hex encoded string with a `0x` prefix)
- Arrays of any of these types
- Objects of any of these types

FireFly will perform basic type checking to make sure the input provided when invoking a contract function can safely be mapped to the FireFly type specified in the FFI. For example, if an FFI defines a function that takes an `integer`, and someone attempts to invoke that function via FireFly's API, if they pass in the JSON `number` of `0.1`, FireFly will reject the request. `0.1` cannot be converted to an integer without loss in precision, so the request will be rejected.

# Drawbacks

[drawbacks]: #drawbacks

While there are many advantages to having this functionality, there are not many drawbacks. Other than time and complexity to implement, there should be no negative impact on existing functionality in FireFly.

# Rationale and alternatives

[alternatives]: #alternatives

This design was chosen because it offers end users the most flexibility in interacting with their custom smart contracts. During the design and brainstorming process, several different ways of invoking a function on a smart contract were considered, and this design enables all of the following:

- Directly invoking a smart contract while passing the schema and contract location / ledger inline with the request
- Uploading an FFI document describing the interface of a custom smart contract
  - Invoking methods on that smart contract while specifying the contract location / ledger inline with the request
- Registering a named API path for using an defined contract interface
  - Optionally specifying a contract location / ledger when creating the named API, or inline with each function invocation request
  - Auto-generated OpenAPI3 YAML documentation for the generated API
  - Allowing the API to be updated to point to a new FFI in the future

# Prior art

[prior-art]: #prior-art

The Ethereum ABI format has strongly influenced the design of the FFI format. This should make the code fairly simple which will translate an existing ABI into an FFI.

# Testing

[testing]: #testing

A new suite of End-to-End tests needs to be included in this work. The suite should cover:

- Directly invoking contracts
- Subscribing to and receiving events for a direct invocation
- Uploading a custom FFI
- Invoking a method on that FFI
- Subscribing to and receiving events for that FFI
- Creating a named API that points to that FFI
- Invoking a method on the named API
- Subscribing to and receiving events for that named API

# Dependencies

[dependencies]: #dependencies

- ~~Ethconnect subscription API improvements~~
- Fabconnect query API

# Unresolved questions

[unresolved]: #unresolved-questions

- No known unresolved questions at this time
