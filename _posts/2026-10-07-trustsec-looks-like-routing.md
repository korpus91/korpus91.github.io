---
layout: post
title: "When TrustSec looks like a routing problem"
---
The ticket said the domain controllers were down. They were not. They answered DNS, Kerberos and LDAP all day. They just would not answer a ping, and every monitoring check, every engineer's first test and half the troubleshooting runbooks start with a ping. Two hours of looking at routing tables later, the cause turned out to be one cell in a TrustSec policy matrix.

## How it happens

TrustSec enforces policy between Security Group Tags (SGTs), not between IP addresses. Every packet gets a source tag; the destination gets a tag from an IP-to-SGT mapping, from the authorization result of the port it sits on, or, if nothing classifies it, the Unknown tag (SGT 0). The switch then applies whatever SGACL sits in the matrix cell for that source and destination pair.

Servers are the classic gap. Users and endpoints get tagged at authentication, but servers often connect to ports that never authenticate, or sit behind a routed hop where no IP-to-SGT mapping was ever created. They end up as Unknown, or under a broad catch-all tag shared with things nobody thought about.

Now add a reasonable-looking policy: someone writes an SGACL for that destination tag that permits the application ports and ends with a deny for everything else, or explicitly denies ICMP to keep scanners quiet. Every real service keeps working. Ping dies. The symptom points straight at Layer 3, and the cause is two layers of policy away from where anyone is looking.

## How to prove it in minutes

The switch keeps per-SGT-pair counters, and they settle the question faster than any trace:

```
show cts role-based sgt-map <server-ip>
show cts role-based permissions to <destination-sgt>
show cts role-based counters to <destination-sgt>
```

The first tells you which tag the server actually has, which is often not the one you assumed. The second shows the SGACL in the cell. The third shows deny counters climbing while you ping. If the deny counter moves in step with your failed pings, routing is innocent.

On the ISE side, the policy matrix view for that destination tag shows the same cell and who last changed it.

Command options differ across IOS-XE releases, so check the exact syntax for your version.

## How to stop it happening again

1. Classify servers deliberately. Static IP-to-SGT mappings or subnet-to-SGT bindings for server ranges, pushed from ISE, so no server is accidentally Unknown.
2. Decide on ICMP as policy, not as a side effect. Permitting echo for troubleshooting into infrastructure tags is a small cost for a large gain in diagnosability.
3. Put the per-pair counters into the troubleshooting runbook, right after "can you ping it". The first question for a segmented network is not "is it routed" but "what tag is it, and what does the cell say".
4. Review matrix changes like firewall changes. A single cell edit has the blast radius of a firewall rule and deserves the same change control.

## The broader lesson

Segmentation moves part of the forwarding decision out of the routing table and into policy. Once that happens, "it does not ping" stops being a Layer 3 symptom by default. Teams that adopt TrustSec need to update their mental model, and their first-response checklist, to match.
