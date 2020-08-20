---
title: Azure NSG Rule Shenanigans
excerpt: Strange things afoot with Azure Network Security Group Rules
category:
  - blog
header:
  image: /images/header-brickwall.png
  overlay_image: /images/header-brickwall.png
  overlay_filter: 0.5
  teaser: /images/teaser-brickwall.png
tags:
  - nsg
  - azure
---

## What are NSG Rules?

Azure provides firewall type features using [Network Security Groups](https://docs.microsoft.com/en-us/azure/virtual-network/security-overview) (NSG). So you can specify:

> Allow TCP Network traffic from Computer A to Port 443 on Computer B

or

> Deny all traffic

I was at a client one day and a colleague asked for my help on a strange problem ...

## Infrastructure Setup

He had a client application that connected to an [F5 Service](https://www.f5.com/products/big-ip-services/virtual-editions) and that then connected to a Windows server in a virtual pool. On the, Windows Server was an NSG applied to the Network Interface. The application connected over TCP Port 9000. So something like this

``` text
                                                          +---+
+--------+                   +--------+                   | N |  +----------------+
| Client |  ---TCP 9000--->  |   F5   |  ---TCP 9000--->  | S |--| Windows Server |
+--------+                   +--------+                   | G |  +----------------+
                                                          +---+
```

But here was the strange thing ... When he applied a rule which allowed all traffic from the F5 to the Windows Server, the application worked correctly. But when he applied a rule which allowed traffic from the F5 to the Windows Server to port 9000 the application didn't work. It didn't make sense! The traffic was allowed. Even the network tracing logs showed the required traffic was on port 9000. So why did adding port 9000 break the application?

## What went wrong?

Looking at the F5 Virtual Pool information, it showed that the pool was unavailable. But the server was clearly available. So why did the F5 think the server wasn't functioning? The virtual pool was using ICMP pings to check if a server was functioning. So the network looked more like this now:

``` text
                                                          +---+
+--------+                   +--------+  ---TCP 9000--->  | N |  +----------------+
| Client |  ---TCP 9000--->  |   F5   |                   | S |--| Windows Server |
+--------+                   +--------+  ---  ICMP  ----  | G |  +----------------+
                                                          +---+
```

The NSG rules should've been allowing all of that traffic, but obviously it was not. We added an additional rule which allowed ICMP from the F5 to the Windows Server, and almost immediately the F5 could ping the server, the virtual pool was available and the appliction worked! Great news, but why did we need the extra rule?

## When ANY doesn't mean ANY

Looking back to the NSG rule we used, it made sense why it broke, but it was certainly not obvious.

The original rule `Allow ANY protocol FROM F5 to WINDOWS SERVER` would allow TCP,ICMP and UDP (The protocols the NSG understands) from the F5 to the Windows Server.

The new rule `Allow ANY protocol FROM F5 to WINDOWS SERVER on Port 9000` was ... interesting. The first part `ANY protocol` would allow TCP, UDP and ICMP. The last part `Port 9000` would allow traffic to UDP and TCP Port 9000. But what about ICMP? Well ICMP doesn't have ports so it can't really ever equal 9000. So the rule was a little confusing. But it turns out any protocol doesn't always _mean_ any.

### Never forget ICMP

So when adding ports to an NSG rule, remember that it won't match the ICMP protocol. For allow or deny rules. And if you need that for logging or health checks, remember to add it in!
