- Feature Name: operation_resubmission
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

Operations in FireFly are stateful external actions triggered via plugins. They can succeed or fail,
or sometimes may hang in a pending state indefinitely (if expected feedback is not received).
Failed or "stale" pending operations will now support resubmission, in order to facilitate recovery
from transient issues.

# Motivation
[motivation]: #motivation

FireFly is largely an orchestration layer to communicate with many backend services, such as blockchains
and data exchange systems. Most of these external communications take place via HTTP or similar network
requests, and as such are inherently unreliable and prone to occasional failures.

FireFly in turn is designed to serve as a backend to some client application. In the case that the client
reaches FireFly but FireFly fails to reach one of its own backend services, it can be very difficult for
the client application to recreate and retry the failed portion of the request. Therefore, FireFly should
track failures itself and allow simple, manual retries on a per-operation basis.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

External actions triggered by FireFly against other backend services - particularly those that change
system state and/or provide asynchronous feedback - are tracked as "operations". Operations record the
inputs and outputs of the action, the targeted plugin, and the current state (Pending/Succeeded/Failed).

Operations can be listed or queried individually with the APIs:
* `GET /namespaces/{ns}/operations`
* `GET /namespaces/{ns}/operations/{opid}`

Any operation may be re-submitted with the API:
* `POST /namespaces/{ns}/operations/{opid}/retry`

It is recommended that operations only be retried if they are in "Failed" state, or are determined to be
stale (such as operations left in "Pending" state with a last update more than 30 minutes in the past).
However, no restrictions are enforced by the backend on what may be retried.

Retrying an operation will create a new copy of the operation, initialized to "Pending" state, and will
re-trigger the plugin action with the same inputs as the original request. The original operation
will be updated with a reference to the new operation.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Operations manager

This change introduces a new component called the `Operations Manager`. This manager will know how to
map a given `Operation` object to an actual plugin call (by inspecting the operation's `Type` and `Input`).
For all plugin calls wrapped by an `Operation`, callers will now _not_ invoke the plugin directly, but
will simply construct the `Operation` object and pass it to the `Operations Manager` for handling the
plugin call. This will also enable the manager to retry any previous operation by simply reading the
`Operation` object and triggering the relevant plugin call again.

## Operation APIs and data model changes

A new API will be added for `POST /namespaces/{ns}/operations/{opid}/retry`. It takes no arguments or body.

The `Operation` type will add a new field for "retry", which is a UUID reference to another operation
that represents a retry of the original. A given operation may only be retried once - but the retry may
be retried if needed (and so on), forming a chain of retried operations.

## Transaction status

The processing for `GET /namespaces/{ns}/transactions/{txnid}/status` will be updated as follows:
* If any child operation is found with status "Failed" _and no retry_, the transaction is "Failed"
* If any child operation is found with status "Pending" _and no retry_, the transaction is "Pending"
* Otherwise, continue with the usual logic for examining blockchain events and other aspects of the transaction

This change also increases the likelihood that a transaction will have a large number of operations
associated with it (in the case of many retries). For efficiency, the status reporting logic should be enhanced
with more targeted queries on the conditions above, rather than relying on listing every single operation.

The `TransactionStatus` type will add a new field for "updated", which is the timestamp of the newest update
to a child in the transaction (operation, blockchain event, etc). This will make it easier to flag "stale"
transactions whose operations are hung.

# Drawbacks
[drawbacks]: #drawbacks

There is little risk associated with this change.

Due to the efficiency considerations noted above, there will be some reduction in the details reported by
`GET /namespaces/{ns}/transactions/{txnid}/status`, which will need to be taken into account for the UI or
anything else that queries this data.

# Rationale and alternatives
[alternatives]: #alternatives

TBD

# Prior art
[prior-art]: #prior-art

TBD

# Testing
[testing]: #testing

New E2E tests will need to be crafted for the retry functionality.

The UI will need to be manually inspected (and adjusted) to ensure that retried operations
are indicated in a clear manner.

# Dependencies
[dependencies]: #dependencies

This is closely tied to FIR #8 and a continuation of much of the work surrounding Transactions
and Operations that has already been committed as part of that item.

# Unresolved questions
[unresolved]: #unresolved-questions

TBD
