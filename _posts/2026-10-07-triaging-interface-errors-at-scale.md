---
layout: post
title: "Triaging interface errors across a whole switch stack"
---
Most physical-layer problems announce themselves in `show interfaces` long before a user opens a ticket. The trouble is volume: a stack of four 48-port switches is close to two hundred interfaces, and nobody reads two hundred counter blocks carefully. Here is the triage order I use, and the tool I wrote to do the reading for me.

## Read the counters as symptoms, not numbers

**CRC errors** mean frames arrived corrupted. On a healthy copper or fiber link the count stays at zero. A handful after a reboot or a cable move is noise; a count that keeps climbing is a physical problem until proven otherwise: the patch cable, the optic, a damaged panel port, or interference. I judge CRCs as a rate against input packets rather than as a raw number, because 50 CRCs on a port that moved a billion packets is a different story from 50 on a port that moved ten thousand.

**Late collisions** on a full-duplex network should not exist. When you see them, or see an up interface running half duplex at all, the classic cause is a duplex mismatch: one side hard-coded, the other auto-negotiating. The fix is symmetry, auto on both ends or the same fixed setting on both ends.

**Runts and giants** point two ways. Runts usually travel with duplex problems or a failing NIC. Giants usually mean an MTU mismatch between neighbors.

**Output drops** are a capacity question, not a cabling one. A tiny fraction is normal on a busy uplink; a sustained rate points at congestion, queueing policy or microbursts.

**Interface resets** climbing over time suggest a flapping link. Check the logs for up and down events before touching anything.

**err-disabled** means the switch shut the port on purpose: port security, BPDU guard, storm control or link flap detection. `show interfaces status err-disabled` names the reason. Fix the cause before you bounce the port, or it will go right back down.

## Let a parser do the reading

I wrote [cisco-interface-health](https://github.com/korpus91/cisco-interface-health) to apply those rules to raw output. It parses `show interfaces` text offline, never connects to a device, and ranks findings by severity with a suggested next step:

```
pip install cisco-interface-health
ifhealth show_interfaces.txt --csv findings.csv
```

Because it only reads text, it is safe to run on output a client emails you, and the CSV drops straight into a ticket or a report. Thresholds sit at the top of the script so you can tune them for your environment.

## Then work the layers

The ranked list tells you where to look, not what to conclude. Clear counters, wait, and re-collect before calling anything fixed. A counter that stops climbing after a cable swap is evidence; a counter you reset and never re-checked is not.