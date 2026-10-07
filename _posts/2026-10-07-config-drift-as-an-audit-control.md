---
layout: post
title: "Configuration drift as an audit control, not a chore"
---
Every network I have inherited had changes nobody could explain. An ACL line added during an outage and never removed. A trunk carrying a VLAN that was supposed to be retired. A port left in a test VLAN. None of these are dramatic, and all of them are how segmentation quietly erodes.

## Why auditors care

Configuration control is not an abstract best practice. It appears directly in the frameworks most clients answer to: NIST SP 800-53 CM-3 (configuration change control) and CM-6 (configuration settings), CIS Control 4 (secure configuration of enterprise assets), and for industrial environments IEC 62443-3-3 SR 7.6 on network and security configuration settings. Each one asks the same question in different words: can you show that what is running is what you approved?

## The minimum viable control

You do not need a commercial platform to answer that question. You need three things:

1. **An approved baseline** of every device's running configuration, captured at a known-good point, ideally right after a change window closes.
2. **A scheduled comparison** of the live configuration against that baseline, with volatile lines (timestamps, byte counts, clock values) filtered so only real changes surface.
3. **A record** of every comparison, including the clean ones, because "we checked and nothing changed" is evidence too.

That is what [cisco-config-drift](https://github.com/korpus91/cisco-config-drift) does. It sends exactly one command to each device, `show running-config`, keeps credentials out of files, writes a unified diff for anything that changed, and logs a line per device per run. Exit codes make it easy to alert from Task Scheduler, cron or a pipeline.

## Read-only on purpose

The tool never pushes configuration, and that is a design choice, not a missing feature. A drift detector that can also remediate is a change tool, and a change tool needs change control of its own. Keeping detection read-only means it can run on a schedule against production, including OT networks where an unplanned change can stop a process, without becoming a risk itself.

## What to do with drift

Every diff ends in one of two places: it is a legitimate change that missed the paperwork, so you approve it and re-baseline, or it is not, so you roll it back and find out how it happened. Either answer is better than not knowing.