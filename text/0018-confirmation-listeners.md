---
layout: default
title: Blockchain confirmation listeners
nav_order: 3
---

- Feature Name: Blockchain confirmation listeners
- Start Date: 8th May 2024
- FIR PR: (leave this empty)
- FireFly Component: (fill me in with underlined FireFly component, core, orderer/consensus and etc.)
- FireFly Issue: (leave this empty)

# Summary

Currently in FireFly you can:
- Listen for a reliable stream blockchain events emitted from successfully executed smart contract logic.
- Get a success/failure event from an `operation` that you submit via your particular FireFly server

However, there is a common pattern of programming that is not well served by this:
- Listen for all confirmed transactions (maybe matching some criteria)
- Query the events associated with that receipt
- Determine whether you want to process the events associated with that receipt

This requires a reliable ordered stream of receipts that are delivered in the order they are **confirmed** on the blockchain (the block containing them pass a point of finality), and the ability to decode the events associated with that receipt.

This issue proposes we:
- [ ] Design+add this new capability to the FFTM connector toolkit, as an optional feature a connector can implement
- [ ] Implement this feature in EVMConnnect
- [ ] Design+add a new `Blockchain confirmations` collection into FireFly core with appropriate listening/filtering
- [ ] Design+add a new semantic for decoding blockchain events on-demand into FireFly core

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in FireFly and you were
teaching it to another FireFly developer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how FireFly programmers should *think* about the feature, and how
  it should impact the way they use FireFly. It should explain the impact as
  concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or
  migration guidance.
- If applicable, describe the differences between teaching this to established
  and new FireFly developers.
- If applicable, describe any changes that may affect the security of
  communications or administration.

For implementation-oriented FIRs (e.g. for validator internals), this section
should focus on how contributors should think about the change, and give
examples of its concrete impact. For policy FIRs, this section should provide
an example-driven introduction to the policy, and explain its impact in
concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the FIR. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this? Are there any risks that must be considered along with
this FIR. 

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus, global state, transaction processors, and smart contracts
  implementation proposals: Does this feature exists in other distributed
  ledgers and what experience have their communities had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other distributed ledgers, provide readers of your FIR with
a fuller picture.  If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if it is an adaptation.

Note that while precedent set by other distributed ledgers is some motivation,
it does not on its own motivate an FIR.  Please also take into consideration
that FireFly sometimes intentionally diverges from common distributed
ledger/blockchain features.

# Testing
[testing]: #testing

- What kinds of test development and execution will be required in order
to validate this proposal, beyond the usual mandatory unit tests?
- List integration test scenarios which will outline correctness of proposed functionality.

# Dependencies
[dependencies]: #dependencies

- Describe all dependencies that this proposal might have on other FIRs, known JIRA issues,
Hyperledger FireFly components.  Dependencies upon FIRs or issues should be recorded as 
links in the proposals issue itself.

- List down related FIRs proposals that depend upon current FIR, and likewise make sure 
they are also linked to this FIR.

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the FIR process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this FIR that could be
  addressed in the future independently of the solution that comes out of this
  FIR?
