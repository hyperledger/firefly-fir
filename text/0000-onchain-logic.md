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

The Custom On-Chain Logic introduces a new set of unified APIs for FireFly to invoke any smart contract and subscribe to events emitted from that contract. It also introduces a consistent way to describe the interface of a smart contract, including its functions, parameters, and events, in a way that is abstracted from a specific blockchain implementation.

> Note: Deployment of smart contracts is not in scope of this improvement request.

# Motivation

[motivation]: #motivation

Many FireFly users have requested the ability to interact with custom smart contracts through FireFly. This is one of the major features planned for v1.0. Right now, custom smart contracts can be invoked directly through FireFly's blockchain connectors, but these APIs are specific to each blockchain connector. Today, there is no way to change blockchain implementations without also changing API calls to interact with a custom contract. The goal of this feature is to have a set of APIs in the FireFly Core which allow users to swap out FireFly plugins such as the blockchain connector, without needing to change any API calls made to FireFly's API.

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

With this enhancement, FireFly exposes a simple set of APIs to work with custom smart contracts. Given a smart contract on a blockchain, these APIs will allow an end user to:

1. Directly invoke or query a smart contract while providing all required schema and type information in a single request.

   | Method | Endpoint                            | Description                                  |
   | ------ | ----------------------------------- | -------------------------------------------- |
   | `POST` | `/namespaces/{ns}/contracts/invoke` | Invoke a method on a smart contract          |
   | `POST` | `/namespaces/{ns}/contracts/query`  | Query a smart contract (read-only operation) |
   | `GET`  | `/namespaces/{ns}/contracts/query`  | Query a smart contract (read-only operation) |

1. Subscribe to events emitted from smart contracts while providing all required schema and type information in a single request. Standard CRUD style REST operations can be used on this endpoint to manage subscriptions as well.

   | Method   | Endpoint                                        | Description           |
   | -------- | ----------------------------------------------- | --------------------- |
   | `POST`   | `/namespaces/{ns}/contracts/subscriptions`      | Create a subscription |
   | `GET`    | `/namespaces/{ns}/contracts/subscriptions`      | List subscriptions    |
   | `GET`    | `/namespaces/{ns}/contracts/subscriptions/{id}` | Get a subscription    |
   | `DELETE` | `/namespaces/{ns}/contracts/subscriptions/{id}` | Delete a subscription |

1. Describe the interface of the smart contract including its functions and events in a standardized format. This format an FFI (FireFly Interface). Contract interfaces require a name and version that is unique within their namespace. When a new contract is defined, it is also sent as a broadcast to other members of the network.

   | Method | Endpoint                                     | Description                                                                            |
   | ------ | -------------------------------------------- | -------------------------------------------------------------------------------------- |
   | `POST` | `/namespaces/{ns}/contracts/interfaces`      | Define a new smart contract interface and broadcast it to other members of the network |
   | `GET`  | `/namespaces/{ns}/contracts/interfaces`      | List contract interfaces                                                               |
   | `GET`  | `/namespaces/{ns}/contracts/interfaces/{id}` | Get a contract interface                                                               |

1. Invoke or query a smart contract that implements a given interface while providing its location information (contract address or channel/chaincode, etc.) inline in the request

   | Method | Endpoint                                            | Description                                  |
   | ------ | --------------------------------------------------- | -------------------------------------------- |
   | `POST` | `/namespaces/{ns}/contracts/interfaces/{id}/invoke` | Invoke a method on a smart contract          |
   | `POST` | `/namespaces/{ns}/contracts/interfaces/{id}/query`  | Query a smart contract (read-only operation) |
   | `GET`  | `/namespaces/{ns}/contracts/interfaces/{id}/query`  | Query a smart contract (read-only operation) |

1. Subscribe to events emitted from a smart contract that implements a given interface while providing its location information (contract address or channel/chaincode, etc.) inline in the request. Standard CRUD style REST operations can be used on this endpoint to manage subscriptions as well.

   | Method   | Endpoint                                                        | Description           |
   | -------- | --------------------------------------------------------------- | --------------------- |
   | `POST`   | `/namespaces/{ns}/contracts/interfaces/{id}/subscriptions`      | Create a subscription |
   | `GET`    | `/namespaces/{ns}/contracts/interfaces/{id}/subscriptions`      | List subscriptions    |
   | `GET`    | `/namespaces/{ns}/contracts/interfaces/{id}/subscriptions/{id}` | Get a subscription    |
   | `DELETE` | `/namespaces/{ns}/contracts/interfaces/{id}/subscriptions/{id}` | Delete a subscription |

1. Register a set of named API endpoints that allow a user to interact with a defined smart contract interface. This will also generate an OpenAPI document that describes how to use each endpoint. Optionally, the location information (contract address or channel/chaincode, etc.) of the contract can part of the API registration request, or it can be specified inline with each invoke/query request.

   | Method   | Endpoint                                                    | Description                                                       |
   | -------- | ----------------------------------------------------------- | ----------------------------------------------------------------- |
   | `POST`   | `/namespaces/{ns}/contracts/apis`                           | Create a named API                                                |
   | `GET`    | `/namespaces/{ns}/contracts/apis`                           | List named APIs                                                   |
   | `GET`    | `/namespaces/{ns}/contracts/apis/{name}`                    | Get a named API                                                   |
   | `PUT`    | `/namespaces/{ns}/contracts/apis/{name}`                    | Update a named API - can be used to update the contract interface |
   | `POST`   | `/namespaces/{ns}/contracts/invoke/{methodName}`            | Invoke a method on a smart contract                               |
   | `POST`   | `/namespaces/{ns}/contracts/query/{methodName}`             | Query a smart contract (read-only operation)                      |
   | `GET`    | `/namespaces/{ns}/contracts/query/{methodName}`             | Query a smart contract (read-only operation)                      |
   | `POST`   | `/namespaces/{ns}/contracts/apis/{name}/subscriptions`      | Create a subscription                                             |
   | `GET`    | `/namespaces/{ns}/contracts/apis/{name}/subscriptions`      | List subscriptions                                                |
   | `GET`    | `/namespaces/{ns}/contracts/apis/{name}/subscriptions/{id}` | Get a subscription                                                |
   | `DELETE` | `/namespaces/{ns}/contracts/apis/{name}/subscriptions/{id}` | Delete a subscription                                             |

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
          "type": "int"
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
          "type": "int"
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
          "type": "string"
        },
        {
          "name": "value",
          "type": "int"
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
  "name": "simple-storage"
}
```

### Looking up a contract instance

```json
[
  {
    "id": "33bcdf7e-3fbe-4ea9-99db-4219f4034242",
    "namespace": "default",
    "contract": {
      "id": "933db039-8240-4bd7-b262-1a92f13ed0a2"
    },
    "ledger": null,
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
  "params": {
    "newValue": 120
  }
}
```

# Drawbacks

[drawbacks]: #drawbacks

There are no known drawbacks at this time.

# Rationale and alternatives

[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art

[prior-art]: #prior-art

The Ethereum ABI format may inform part of the design for FireFly's Contract Definition format.

# Testing

[testing]: #testing

- What kinds of test development and execution will be required in order
  to validate this proposal, beyond the usual mandatory unit tests?
- List integration test scenarios which will outline correctness of proposed functionality.

# Dependencies

[dependencies]: #dependencies

- Ethconnect subscription API improvements
- Fabconnect query API

# Unresolved questions

[unresolved]: #unresolved-questions

- Nested object types in FFI
- Arrays of types in FFI
