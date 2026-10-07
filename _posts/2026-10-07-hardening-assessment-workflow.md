---
layout: post
title: "A network hardening assessment in two read-only steps"
---
Clients ask for a hardening review in one of two moods: before an audit, or after a scare. Either way they want the same deliverable: what is weak, how bad it is, and exactly what to change. Here is the workflow I use, built so that nothing touches production except one read-only command.

## Step 1: collect, read-only

Pull every running-config with a single `show running-config` per device. [cisco-config-drift](https://github.com/korpus91/cisco-config-drift) does this as its snapshot step, so the collection doubles as the client's first approved baseline:

```
pip install cisco-config-drift
cisco-drift snapshot --inventory inventory.csv
```

Credentials come from environment variables or a prompt and never touch disk. If the client prefers to collect configs themselves, they can, and nothing in step 2 changes.

## Step 2: audit, offline

Run [ios-hardening-audit](https://github.com/korpus91/ios-hardening-audit) across the collected configs:

```
pip install ios-hardening-audit
ios-harden "baselines/*.cfg" --csv findings.csv
```

It checks 23 baseline items drawn from CIS Cisco IOS benchmarks and vendor hardening guides: Telnet on vty lines, missing access-class, `enable password` and weakly hashed secrets, local users stored as passwords, default or read-write SNMP communities, the HTTP server, missing AAA, remote syslog, NTP, banners and brute-force protection. Every finding carries the evidence line and the fix, and secrets are redacted from the output so the report can be shared.

## Step 3: turn findings into a plan, not a list

A raw list of two hundred findings is not a deliverable. What the client needs is:

1. **The handful of highs, by device.** Telnet, default communities and cleartext credentials are the ones attackers actually use. These go first, usually in one change window.
2. **Fleet-wide patterns.** If forty switches share the same missing `access-class`, that is one template fix, not forty tickets.
3. **Version caveats.** Some defaults differ by IOS release, so every "not found in config" finding is confirmed against the running version before it goes in the report.
4. **A re-run.** After remediation, the same two commands produce the after picture. The diff between the two reports is the evidence the auditor wants.

## Why read-only matters

Assessments that start by pushing an agent or a config change need their own change approval, and that alone can delay the work by weeks. A read-only collection plus an offline audit can usually start the same day, and it can run safely against OT and other networks where an unplanned change is not an option.