---
layout: default
title: FIR Template
nav_order: 3
---

- Feature Name: Rest API Query Syntax Enhancements
- Start Date: Jan 5th 2022
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary
[summary]: #summary

Update the REST API query syntax, to incorporate "starts with" and "ends with",
while addressing some confusions with the current system.

# Motivation
[motivation]: #motivation

- We need to implement "starts with" queries, for UI function to perform
  tree-based navigation of Data objects. Makes sense to implement
  "ends with" at the same time.
- We've spent the symbols `^` & `@` for "containing" leaving nothing that
  makes sense to use as starts-with or ends-with.
- The use of different symbols for case-sensitive vs. case-insensitive
  compare is confusing
- There is no way to perform a case-insensitive exact equals
- There is no way to specify greater than `"=something"`

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The current rules are as follows:

| Operator | Description                       |
|----------|-----------------------------------|
| (none)   | Equal                             |
| `!`      | Not equal                         |
| `<`      | Less than                         |
| `<=`     | Less than or equal                |
| `>`      | Greater than                      |
| `>=`     | Greater than or equal             |
| `@`      | Containing - case sensitive       |
| `!@`     | Not containing - case sensitive   |
| `^`      | Containing - case insensitive     |
| `!^`     | Not containing - case insensitive |

Proposed rules are as follows:

### Query syntax

`field=[modifiers][operator][match-string]`


### Operators

Operators are a type of comparison operation to
perform against the match string.

| Operator | Description                        |
|----------|------------------------------------|
| `=`      | Equal                              |
| (none)   | Equal (shortcut)                   |
| `@`      | Containing                         |
| `^`      | Starts with                        |
| `$`      | Ends with                          |
| `<<`     | Less than                          |
| `<`      | Less than (shortcut)               |
| `<=`     | Less than or equal                 |
| `>>`     | Greater than                       |
| `>`      | Greater than (shortcut)            |
| `>=`     | Greater than or equal              |

> Shortcuts are only safe to use when your match
> string could never start with a character outside of
> `a-z`, `A-Z`, `0-9`, `-` or `_`.

### Modifiers

Modifiers can appear before the operator, to change its
behavior.

| Modifier | Description                        |
|----------|------------------------------------|
| `!`      | Not - negates the match            |
| `:`      | Case insensitive                   |
| `?`      | Treat empty match string as null   |

| Example      | Description                                |
|--------------|--------------------------------------------|
| `cat`        | Equals "cat"                               |
| `=cat`       | Equals "cat" (same)                        |
| `!=cat`      | Not equal to "cat"                         |
| `:=cat`      | Equal to "CAT", "cat", "CaT etc.           |
| `!:cat`      | Not equal to "CAT", "cat", "CaT etc.       |
| `=!cat`      | Equal to "!cat" (! is after operator)      |
| `^cats/`     | Starts with "cats/"                        |
| `$_cat`      | Ends with with "_cat"                      |
| `!:^cats/`   | Does not start with "cats/", "CATs/" etc.  |
| `!$-cat`     | Does not end with "-cat"                   |
| `?=`         | Is null                                    |
| `!?=`        | Is not null                                |

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The above rules are considered an improvement, because they:
- Solve the specific limitations specified in the motivations
- Do not reserve symbols that are likely to have other meanings in the future
  - `|` - strong ties to "or" functionality
  - `~` - strong ties to Regular Expression functionality
  - `*` - strong ties to wildcard functionality
  - `-` - commonly used as a character in match strings
  - `_` - commonly used as a character at beginning/end of match strings
- Are inexpensive to parse - you just stop processing modifiers after
  you see the first operator character, or character that is neither an
  operator or a modifier.

# Drawbacks
[drawbacks]: #drawbacks

There is a breaking change in this PR to existing API queries. Specifically
the `^` character has been repurposed, and there are subtle changes in
how existing search strings would be treated that contain special characters.

This proposal still only solves "simple" queries, that can be expressed via
a flat set of un-grouped match fields. However, this proposal does not
preclude the future introduction of a `query=` special query parameter
that can contain a DSL (JSON, or otherwise) for nested/grouped queries.

# Rationale and alternatives
[alternatives]: #alternatives

One key alternative to the above, would be supporting generic wildcard or
regular expression syntax, instead of "starts with", "ends with", "contains"
semantics. This was eliminated at this time, as it places to high a burden
on the pluggable database implementations to support a rich filtering syntax.

We have already in FireFly built all the infrastructure to implement a DSL, but exposing
that with full nesting etc. encourages a use of complex queries that are not well
indexed. The goal today is to provide "just enough" to allow the majority of
FireFly Explorer UI functionality to work effectively, and to allow applications to
query the data they need in a simple self-documenting way.

This FIR does not want to deviate heavily or provide a high migration impact for API
users. Although a small migration impact to allow us to reclaim the `^` operator for
"starts with" is proposed as worthwhile.

# Prior art
[prior-art]: #prior-art

There are various query systems available on REST APIs.

A few examples known to the author of this FIR, with the following representing
the kind of progression seen from quick-and-easy, through to full DSL:
- https://www.npmjs.com/package/mongo-querystring
- https://github.com/nestjsx/crud/wiki/Requests#filter
- https://loopback.io/doc/en/lb3/Where-filter.html

There does not seem to be any one well adopted standard.

# Testing
[testing]: #testing

Migration of the UI filter explorer and exploratory testing of various filters is
planned, in addition to the normal UT and E2E coverage.

# Dependencies
[dependencies]: #dependencies

None

# Unresolved questions
[unresolved]: #unresolved-questions

None
