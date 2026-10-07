---
layout: post
title: "The ISE certificate that takes 802.1X down on a Tuesday"
---
Certificate expiry is the most predictable outage in network access control, and it still catches teams every year. The expiry date has been printed on the certificate since the day it was issued. What fails is the process around it.

## What breaks, and how

Cisco ISE carries several system certificates per node, each bound to one or more roles. Two of them matter most:

**The EAP certificate** is what ISE presents to supplicants during EAP-TLS, PEAP and TEAP. When it expires, endpoints that validate the server certificate (as they should) refuse to continue. Wired and wireless 802.1X authentications start failing, and depending on your fallback design, devices land in a remediation VLAN or nowhere at all. Endpoints that already hold a session may keep working until reauthentication, which makes the outage roll in over hours rather than all at once and makes it harder to diagnose.

**The Admin certificate** secures the management GUI and ISE's own communication between nodes in the deployment. Replacing it restarts services on that node, so the renewal itself needs a change window, which is exactly why it gets put off.

Trusted certificates expire too. An expired root or intermediate in the trusted store breaks validation of every client certificate chained to it.

## Why alarms are not enough

ISE raises certificate expiration alarms, and in most deployments I have seen they land in a dashboard nobody watches daily or an inbox filter nobody reads. The fix is not more alarms. It is a check that runs on a schedule, produces a short list, and fails loudly.

## A read-only check you can schedule

[ise-health-audit](https://github.com/korpus91/ise-health-audit) uses three GET calls on the ISE OpenAPI to list deployment nodes, their system certificates and the trusted store, then flags anything expired or inside a 30 or 60 day window, plus nodes that are not connected, missing admin HA, and self-signed certificates still serving Admin or EAP:

```
python ise_audit.py --pan ise-pan1.example.com --ca-bundle corp-ca.pem
```

It never writes to ISE. If the client will not grant API access, they can run the three GET calls themselves and hand over the JSON, and the tool audits it offline.

## The process that actually prevents the outage

1. Run the check weekly and route a non-zero exit to a ticket, not an email.
2. Treat the 60 day mark as the start of the renewal, not a reminder: request the certificate, book the window for the Admin role restart, and test the new EAP certificate against each supplicant type before it goes live.
3. After renewal, re-run the check and attach the clean output to the change record.

Sixty days is enough time to do this calmly. Six hours is not.