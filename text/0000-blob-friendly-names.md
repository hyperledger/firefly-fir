---
layout: default
title: Blob Friendly Names
nav_order: 3
---

- Feature Name: Blob Friendly Names
- Start Date: 2021-01-04
- FIR PR: (leave this empty)
- FireFly Component: Core
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

Make a friendly "path/filename.ext" and blob size first class indexed fields
blob data resources. Also add Explorer UI panels that let you filter/sort
on these, including a hierarchical file-system like navigation using the
convention of splitting on `/`. 

# Motivation
[motivation]: #motivation

Document management is a key use case for any blockchain backed solution that
involves business processes, or business transactions. Humans naturally think
of documents by filename, and organize them in folders decades of experience in
computing in file-systems.

Care is needed to accommodate the FireFly architecture for storage and transfer
blobs. This FIR assets that FireFly cannot enforce a requirement to provide a
name for a blob, or a requirement for a name to be unique within a path.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This FIR proposes two new fields on the `Data` resource.

During validation, on input from API or the validation of network data, these
fields are generated. They cannot be set directly, but are persisted in the
database for indexed sorting/searching.

- `blob.name` - JSON `string`
  - A string with a _convention_ to use `path/to/file.ext`
  - Extracted from top-level properties the the JSON `value`.
  - Extracted based on the following rules of precedence
    1. `value.name` is used directly if it exists
    2. `{{value.path}}/{{value.filename}}` is used if both `value.path` and `value.filename` exist
    3. `value.filename` is used directly if it exists
    4. Otherwise the value is an empty string
  - We require this to be stored within the `value` JSON, so that it contributes to
    the data hash. As such two `Data` that are uploaded with the same payload and different
    filenames, will have different `hash` values, but the same `blob.hash`.
- `blob.size` - JSON `number` (`float64`)
  - For API uploads, this is the size of the blob as calculated when performing
    a hash confirmed streaming upload of the data to the local Data Exchange plugin.
  - For data received across the network, this is the size of the blob as
    reported by Data Exchange when it triggered the `BLOBReceived` callback.
  - If the Blob was stored by a Data Exchange that does not report size (such as
    before this feature was implemented), the size might be zero.

Note that the `autometa`form-data upload field on `POST` will automatically
set the `value.filename` to be per the upload.

During download the `blob.size` as well as the `blob.hash` can be checked during
the download verification.

Previous to this FIR, the `value.size` was calculated by `autometa`, and this no
longer makes sense - as it is not validated to be correct.
So the `autometa` code will be changed to remove the `size` field as it is
not useful and potentially confusing.

UI panels designs for the Hyperledger FireFly Explorer UI will be added
to the FIR by @heymrbrian or @lanasta once ready.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

No additional detail provided in this section. Above considered sufficient.

# Drawbacks
[drawbacks]: #drawbacks

- The `value.name`, `value.filename` and `value.path` fields of data take on
a special meaning, per the above rules of precedence.

- The contract between Data Exchange and FireFly Core to report the arrival
of a new data blob must be tweaked slightly to include an ability to report
the size of the received blob.

# Rationale and alternatives
[alternatives]: #alternatives

Evaluated alternatives as follows:
- Specifying the `blob.name` directly, separate to the `value` JSON
  - The drawback is this would not contribute to the hash, without changing
    the hashing rules for Data.
- Allowing `blob.size` to be specified via `value.size`, or requiring them
  to match.
  - The drawback is this would require full access to the payload during
    validation of the data. We have a nice separation of concerns at the moment
    where DX manages the storage of these blobs, and assures the hash.

# Prior art
[prior-art]: #prior-art

No additional detail provided in this section. Above considered sufficient.

# Testing
[testing]: #testing

- e2e test updates to validate propagation of `blob.name` and `blob.size`
  from local to remote FireFly nodes.

# Dependencies
[dependencies]: #dependencies

None

# Unresolved questions
[unresolved]: #unresolved-questions

None