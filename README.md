---
layout: default
title: FIR Process
nav_order: 2
---

# Hyperledger FireFly FIR Process

If you want to add a feature to FireFly, you've come to the right place.

This lightweight process helps you share and track your contribution, where it's as simple as adding
a new field to an API, or as complex as a whole new plugin.

## How to get started

If you want to implement a new FireFly feature, create a PR to this repo copying
[0000-template.md](0000-template.md) to `text/0000-short-feature-name.md`, and filling it out.

Your choice whether you start the process before, during, or after you start coding.

The community doesn't want to put you off, or slow you down. We just want to keep the community
informed, and provide an inclusive way to let the community provide input before new features
are released.

## FAQ

### Where are the rest of the instructions?

You should find everything you need in the template. The maintainers favor discussion over
formal process so come talk to us on Discord!

> @nguyer - not sure how to invite people to discord. Process feels broken without that

### Do I need a FIR for every improvement?

If you're changing the externals of FireFly, then yes please. It shouldn't take long.

If you're just fixing something, or making it work better internally, then a PR
discussion with the maintainers is enough.

### Do I really have to go through a bunch of process, just to submit some code???

We'd like to FIR process light weight enough, that it feels natural to raise one.

If it doesn't tell us, and we'll try and cut out the pain.

The benefit for you, is people get to hear about your work!

New FIRs are automatically announced on the Discord to the community.

### Does my FIR have to close before I can merge my PR?

No - merging to `main` is based on the maintainers' discretion on an individual repo.

Depending on their assessment of risk of it holding up the next minor release, or
needing backing out, they might ask you to progress the FIR during the PR discussion.

Please include `FIR-XYZ:` in your PR descriptions so we know which FIRs they relate to,
and also mention the FIR issue link in your PRs so Github can do its magic.

> Please note that all PRs for _open_ FIRs will be candidates for the next
> minor release (they will not usually be backported to patch releases on the current
> minor release)