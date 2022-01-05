---
layout: default
title: FIR Process
nav_order: 2
---
# Hyperledger FireFly FIR Process

## TL/DR

If you want to implement a new FireFly feature, create a PR to this repo copying
[0000-template.md](0000-template.md) to `text/0000-short-feature-name.md`, and filling it out.

Your choice whether you start the process before, during, or after you start coding.

The community doesn't want to put you off, or slow you down. We just want to keep the community
informed, and provide an inclusive way to let the community provide input before new features
are released.

## Introduction

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal [GitHub pull request workflow](https://guides.github.com/introduction/flow/).

Some changes though are substantial, and we ask that these be put through a bit
of a design process and produce a consensus among the FireFly maintainers and
broader community.

The FIR (FireFly Improvement Request) _(pronounced "fire")_ process is intended to provide a consistent and
controlled path for major changes to FireFly and other official project
components, so that all stakeholders can be confident about the direction in
which FireFly is evolving.

This process is intended to be substantially similar to the RFCs process other
Hyperledger teams have adopted, customized as necessary for use with FireFly.
The `README.md` and `0000-template.md` files were forked from the
[Fabric RFCs repo](https://github.com/hyperledger/fabric-firs) which was in turn forked from the [Sawtooth RFCss repo](https://github.com/hyperledger/sawtooth-firs), which was itself
derived from the Rust project.

## Table of Contents

- [When you need to follow this process]
- [Before creating a FIR]
- [What the process is]
- [The FIR life-cycle]
- [Reviewing FIRs]
- [Implementing a FIR]
- [License]
- [Contributions]

## When you need to follow this process

[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make substantial changes to the
FireFly Core, any of its sub-components including but not limited to, firefly-ethconnect, firefly-fabconnect or the FIR process
itself. What constitutes a substantial change is evolving based on community
norms and varies depending on what part of the ecosystem you are proposing to
change, but may include the following:

- Architectural changes
- Substantial changes to component interfaces
- New core features
- Backward incompatible changes
- Changes that affect the security of communications or administration

Some changes do not require a FIR:

- Rephrasing, reorganizing, refactoring, or otherwise "changing shape does not
change meaning".
- Additions that strictly improve objective, numerical quality criteria
(warning removal, speedup, better platform coverage, more parallelism, trap
more errors, etc.).

If you submit a pull request to implement a new feature without going through
the FIR process, it may be closed with a polite request to submit a FIR first.

## Before creating a FIR

[Before creating a FIR]: #before-creating-a-fir

A hastily-proposed FIR can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected changes, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the FIR can
make the process smoother.

Although there is no single way to prepare for submitting a FIR, it is
generally a good idea to pursue feedback from other project developers
beforehand, to ascertain that the FIR may be desirable; having a consistent
impact on the project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting a FIR include
talking the idea over to the [FireFly mailing list](https://lists.hyperledger.org/g/FireFly/topics).

As a rule of thumb, receiving encouraging feedback from long-standing
project developers, and particularly the project's maintainers is a good
indication that the FIR is worth pursuing.

## What the process is

[What the process is]: #what-the-process-is

In short, to get a major feature added to FireFly, one must first get the FIR
merged into the FIR repository as a markdown file. At that point the FIR is
"active" and may be implemented with the goal of eventual inclusion into
FireFly.

- Fork [the FIR repository](https://github.com/hyperledger/FireFly-fir).
- Copy `0000-template.md` to `text/0000-my-feature.md`, where "my-feature" is
descriptive. Don't assign a FIR number yet.
- Fill in the FIR. Put care into the details — FIRs that do not present
convincing motivation, demonstrate understanding of the impact of the design,
or are disingenuous about the drawbacks or alternatives tend to be
poorly received.
- Submit a pull request. The pull request will be assigned to a maintainer, and
will receive design feedback from the larger community; the FIR author should
be prepared to revise it in response.
- Build consensus and integrate feedback. FIRs that have broad support are much
more likely to make progress than those that don't receive any comments. Feel
free to reach out to the pull request assignee in particular to get help
identifying stakeholders and obstacles.
- The maintainers will discuss the FIR pull request, as much as possible in the
comment thread of the pull request itself. Offline discussion will be
summarized on the pull request comment thread.
- A good way to build consensus on a FIR pull request is to summarize the FIR on a
community contributor meeting. Coordinate with a maintainer to get on a contributor
meeting agenda. While this is not necessary, it may help to foster sufficient
consensus such that the FIR can proceed to final comment period.
- FIRs rarely go through this process unchanged, especially as alternatives and
drawbacks are shown. You can make edits, big and small, to the FIR to clarify
or change the design, but make changes as new commits to the pull request, and
leave a comment on the pull request explaining your changes. Specifically, do
not squash or rebase commits after they are visible on the pull request.
- At some point, a FireFly maintainer will propose a "motion for final comment
period" (FCP), along with a *disposition* for the FIR (merge, close, or
postpone).
  - This step is taken when enough of the tradeoffs have been discussed that
  the maintainers are in a position to make a decision. That does not require
  consensus amongst all participants in the FIR thread (which is usually
  impossible). However, the argument supporting the disposition on the FIR
  needs to have already been clearly articulated, and there should not be a
  strong consensus *against* that position. FireFly maintainers will use their
  best judgment in taking this step, and the FCP itself ensures there is ample
  time and notification for stakeholders to push back if it is made
  prematurely.
  - For FIRs with lengthy discussion, the motion to FCP is usually preceded by
  a *summary comment* trying to lay out the current state of the discussion and
  major trade-offs/points of disagreement.
  - Before actually entering FCP, the FireFly maintainer who proposes that the
  FIR enter FCP ensures that other interested maintainers have reviewed the FIR
  and at least two other maintainers (three total) have indicated agreement;
  this is often the point at which many maintainers first review the FIR in
  full depth. Note that maintainers from any FireFly repository may review and
  indicate agreement, especially for FIRs that impact multiple repositories.
- The FCP lasts one week, or seven calendar days. It is also advertised widely,
e.g. in the [FireFly Mailing List](https://lists.hyperledger.org/g/FireFly/topics).
This way all stakeholders have a chance to lodge any final objections before a
decision is reached.
- In most cases, the FCP period is quiet since the most interested maintainers
have already indicated agreement, and the FIR is either merged or
closed. However, sometimes substantial new arguments or ideas are raised, the
FCP is canceled, and the FIR goes back into development mode.

## The FIR life-cycle

[The FIR life-cycle]: #the-fir-life-cycle

Once a FIR is merged, it becomes "active" and developers may implement it and submit the code
change as a pull request to the corresponding FireFly repo. Being "active" is
not a rubber stamp, and it does not mean the change will ultimately be merged;
it does mean that in principle all the major stakeholders have agreed to the
change, and are amenable to merging it.

Furthermore, the fact that a given FIR has been accepted and is "active"
implies nothing about what priority is assigned to its implementation, nor does
it imply anything about whether a FireFly developer has been assigned the task
of implementing the feature. While it is not *necessary* that the author of the
FIR also write the implementation, it is by far the most effective way to see
a FIR through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted feature.

Modifications to active FIRs can be done in follow-up pull requests. We strive
to write each FIR in a manner that it will reflect the final design of the
feature; but the nature of the process means that we cannot expect every merged
FIR to actually reflect what the end result will be at the time of the next
major release.

In general, once accepted, FIRs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes
should be new FIRs, with a note added to the original FIR. Exactly what counts
as a "very minor change" is up to the maintainers to decide.

## Reviewing FIRs

[Reviewing FIRs]: #reviewing-firs

While the FIR pull request is up, the maintainers may schedule meetings with
the author and/or relevant stakeholders to discuss the issues in greater
detail, and in some cases the topic may be discussed at a contributors meeting.
In either case a summary from the meeting will be posted back to the FIR pull
request.

The FireFly maintainers make the final decisions about FIRs after the benefits
and drawbacks are well understood. These decisions can be made at any time, but
the maintainers will regularly issue decisions. When a decision is made, the
FIR pull request will either be merged or closed. In either case, if the
reasoning is not clear from the discussion in thread, the maintainers will add
a comment describing the rationale for the decision.

## Implementing a FIR

[Implementing a FIR]: #implementing-a-fir

Some accepted FIRs represent vital changes that need to be implemented right
away. Other accepted FIRs can represent changes that can wait until a
developer feels like doing the work. Every accepted FIR has an associated
issue tracking its implementation in the [FireFly JIRA issue tracker](https://jira.hyperledger.org/projects/FAB/issues).

The author of a FIR is not obligated to implement it. Of course, the FIR
author, as any other developer, is welcome to post an implementation for review
after the FIR has been accepted. Use JIRA for this.

## License

[License]: #license

This repository is licensed under [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0)
([LICENSE](LICENSE)).

## Contributions

[Contributions]: #contributions

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall
be licensed as above, without any additional terms or conditions.
