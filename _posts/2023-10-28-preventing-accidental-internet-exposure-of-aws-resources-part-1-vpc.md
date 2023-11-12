---
layout: post
toc: true
title: "[Almost done] Preventing Accidental Internet-Exposure of AWS Resources (Part 1: VPC)"
---

[Many AWS customers have suffered breaches](https://github.com/ramimac/aws-customer-security-incidents#background) due to exposing resources to the Internet by accident, resources that an attacker can find via traditional public IP network scanning or [searching Shodan](https://maia.crimew.gay/posts/how-to-hack-an-airline/). This three-part series walks through different ways to mitigate that risk.

## About The Problem

There are many ways to make resources public in AWS. [github.com/SummitRoute/aws_exposable_resources](https://github.com/SummitRoute/aws_exposable_resources#aws-exposable-resources) was created specifically to maintain a list of all AWS resources that can be publicly exposed and how.

This post discusses preventing public network access for resources exclusively in a VPC (EC2 instances, ELBs, RDS databases, etc.).

Ideally, you can look at your AWS organization structure from a 1000-foot view and know which subtree of accounts / OUs can have publicly accessible VPCs.

What Good Looks Like:
![alt text](https://i.imgur.com/cVFUpkJ.png)

## Solving The Problem

You can implement this by banning `"ec2:CreateInternetGateway"` in subaccounts via SCP.[^2111]

The reason this works is because although there are many ways an accidental Internet-exposure might happen -- for VPCs at least -- every way requires an Internet Gateway (IGW). E.g.
![alt text](https://i.imgur.com/1e4M8z4.gif)

Or:       
![alt text](https://i.imgur.com/gyXZz2E.gif)

[^2111]: Alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.

With IGWs banned, you can hand subaccounts over to customers, and they will never be able to make public-facing load balancers or EC2 instances regardless of their IAM permissions!

There is only one complication with this.

In AWS: <ins>Egress to the Internet is tightly coupled with Ingress from the Internet</ins>. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

They are tightly coupled because both require an Internet Gateway (IGW).

The Egress use-case typically looks like:
![alt text](https://i.imgur.com/vKsdNOh.png)

## Supporting Egress in Private VPC Accounts

To support the Egress use-case, you must ensure your network architecture tightly couples NAT with an Internet Gateway by, e.g., giving subaccounts a paved path to a NAT Gateway in another account. You can do this via:

1. [Centralized Egress via Transit Gateway (TGW)](#option-1-centralized-egress-via-transit-gateway-tgw)
2. [Centralized Egress via PrivateLink (or VPC Peering) with Egress Filtering](#option-2-centralized-egress-via-privatelink-or-vpc-peering-with-egress-filtering)
3. [Centralized Egress via Gateway Load Balancer (GWLB)](#option-3-centralized-egress-via-gateway-load-balancer-gwlb-with-firewall)
4. [VPC Sharing](#option-4-vpc-sharing)
5. [IPv6 for Egress](#option-5-ipv6-for-egress)

Hopefully, one of these options will align with the goals of your networking team.

My recommendation is to go with TGW for most accounts, and VPC Sharing for sandbox accounts. Once you are ready to implement Egress Filtering, use PrivateLink + Egress Filtering.

### Option 1: Centralized Egress via Transit Gateway (TGW)

This is the most common implementation, and probably the best. If money is no issue for you: go this route.

It looks like this:
![alt text](https://i.imgur.com/alRH2hN.png) 

AWS first wrote about this [in 2019](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-a-single-internet-exit-point-from-multiple-vpcs-using-aws-transit-gateway/) and lists it under their prescriptive guidance as [Centralized Egress](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/centralized-egress.html).

As you can see, each VPC in a subaccount has a route table with 0.0.0.0/0 destined traffic sent to a TGW in another account, where an IGW does live.

#### A Note On Cost

This will reduce your NAT Gateway cost, which is arguably the most notoriously expensive networking component in AWS.

On one hand, [AWS states](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-egress-to-internet.html):

>Deploying a NAT gateway in every AZ of every spoke VPC can become cost-prohibitive because you pay an hourly charge for every NAT gateway you deploy, so centralizing could be a viable option.

On the other hand, they also say:

>In some edge cases, when you send huge amounts of data through a NAT gateway from a VPC, keeping the NAT local in the VPC to avoid the Transit Gateway data processing charge might be a more cost-effective option.

Sending huge amounts of data through a NAT Gateway should be avoided anyway.
[S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html), [Splunk](https://www.splunk.com/en_us/blog/platform/announcing-aws-privatelink-support-on-splunk-cloud-platform.html), [Honeycomb](https://docs.honeycomb.io/integrations/aws/aws-privatelink/), and similar companies[^1350] have VPC endpoints you can utilize to lower NAT Gateway data processing charges.


[^1350]: There are some companies with agents meant to be deployed on EC2s that do not offer a VPC endpoint, [perhaps](https://sso.tax/) a [wall](https://fido.fail/) of [shame](https://github.com/SummitRoute/imdsv2_wall_of_shame#imdsv2-wall-of-shame) can be made.


The following is a graph, [generated with Python](https://gist.github.com/MajinBuuOnSecurity/361a0bac8e65432a567d5d157d2524d5). As you can see at e.g. 20 VPCs you'd need to be sending over 55 TB for centralized egress to be more expensive, it only gets more worthwhile the more VPCs you add.
![alt text](https://i.imgur.com/fxKEToY.png)

See [the FAQ](#can-you-walk-through-the-cost-details-around-option-1) for a verbose example.

### Option 2: Centralized Egress via PrivateLink (or VPC Peering) with Egress Filtering

PrivateLink and VPC Peering are mostly non-options. See the [FAQ](#why-is-vpc-peering-not-a-straightforward-option) for more information.

However, if you are willing to do a lot of heavy lifting that is orthogonal to AWS primitives, you _can_ use these in combination with [Internet Egress Filtering](https://eng.lyft.com/internet-egress-filtering-of-services-at-lyft-72e99e29a4d9), to accomplish centralized egress. This is because the destination IP of outbound traffic won't be the Internet, but a private IP.

This is assuming you are deploying e.g. iptables, to re-route Internet-destined traffic on every host.

Some reasons you may not want to do this are:
- Significant effort
- It won't be possible for all subaccount types, such as sandbox accounts. (Where requiring `iptables` and a proxy are too heavyweight.)
- Egress filtering is lower-priority than preventing accidental Internet-exposure. So tightly coupling the two and needing to setup a proxy first may not make strategic sense.
- If something goes wrong on the host, the lost traffic will not appear in VPC flow logs [^98] or traffic mirroring logs.[^985] The DNS lookups will show up in Route53 query logs, but that's it.

With that said, AWS does not have a primitive to perform Egress filtering,[^99] so you will eventually have to implement Egress filtering via a proxy or GWLB. Therefore, in e.g. production accounts, you could go with this option.

Using PrivateLink, in this way, would look like this:
![alt text](https://i.imgur.com/vg9rcTE.png)

(Note that no NAT Gateway is necessary here, as Envoy is running in an public subnet.)

[^98]: [Flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-limitations) does not state "Internet-bound traffic sent to a peering connection" or "Internet-bound traffic sent to a VPC interface endpoint." under `The following types of traffic are not logged:`. After testing I believe these are likely omitted due to not being a proper use-case.

[^985]: A peering connection cannot be selected as a [traffic mirror source or target](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-targets.html), but a network interface can. However, only an ENI belonging to an EC2 instance can be a mirror source, not an ENI belonging to an Interface endpoint. The documentation doesn't mention this anywhere I could find.

[^99]: It has [AWS Network Firewall](https://aws.amazon.com/network-firewall/faqs/), which can be fooled via SNI spoofing. So it is, at best, a stepping stone to keep an inventory of your Egress traffic if you can’t get a proxy up and running short-term and are not using TLS 1.3 with encrypted client hello (ECH) or encrypted SNI (ESNI). I cringe at how [the FAQ](https://aws.amazon.com/network-firewall/faqs/) says these are not supported, rather than a bypass of the product. Sadly this euphemism [isn't unique to AWS](https://i.imgur.com/dPyFaNK.png).

### Option 3: Centralized Egress via Gateway Load Balancer (GWLB) with Firewall

TODO: Re-write this section.

[GWLB](https://aws.amazon.com/blogs/aws/introducing-aws-gateway-load-balancer-easy-deployment-scalability-and-high-availability-for-partner-appliances/) is a service intended to enable the deployment of virtual appliances in the form of firewalls, intrusion detection/prevention systems and deep packet inspection systems. The appliances get sent the original traffic [encapsulated via the Geneve protocol](https://aws.amazon.com/blogs/networking-and-content-delivery/integrate-your-custom-logic-or-appliance-with-aws-gateway-load-balancer/).[^99328]

[^99328]: The usual suspects [Aiden Steele](https://awsteele.com/blog/2022/01/20/aws-gwlb-deep-packet-manipulation.html), [Luc van Donkersgoed](https://web.archive.org/web/20220129101637/https://www.sentiatechblog.com/geneveproxy-an-aws-gateway-load-balancer-reference-application), and [Corey Quinn](https://www.lastweekinaws.com/blog/what-i-dont-get-about-the-aws-gateway-load-balancer/) have written about this feature.

The reason why PrivateLink was mostly a non-option was that interface endpoint ENIs have "[Source/destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics)" enabled, and [you cannot disable it since it is managed by AWS](#why-is-vpc-privatelink-not-a-straightforward-option).

However, Gateway Load Balancer endpoint ENIs _with "[Source/destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics)" disabled!_  This is to accompany their intended use-case of inspecting traffic.

This enables us to send 0.0.0.0/0 destined traffic to a GWLBe, in the same way we did for the TGW in Option 1.

This looks like:
![alt text](https://i.imgur.com/tIQaTa0.png)

(Note that no NAT Gateway is necessary here, as the firewall is running in an public subnet and performing NAT. AWS calls this [two-arm mode](https://aws.amazon.com/blogs/networking-and-content-delivery/best-practices-for-deploying-gateway-load-balancer/).)

Security-wise, unfortunately, unless they do decryption, firewalls can't filter on URL path -- for example, you can't block all of github.com except for github.com/mycompany. The better vendors don't seem to offer decryption, I think you're better off using a proxy (i.e. the previous option) if you want to filter on URL paths.

Because the firewall needs to support Geneve encapsulation, be invulnerable to SNI spoofing, fast, reliable and not [susceptible to IP address mismatches](https://chasersystems.com/discriminat/faq/#are-the-out-of-band-dns-lookups-susceptible-to-ip-address-mismatches). It is not easy to create an open-source alternative to DiscrimiNAT.

DiscrimiNAT seems superior to Palo Alto Firewall[^99351] as you just [edit some security group descriptions to configure it](https://chasersystems.com/docs/discriminat/aws/quick-start/#viii-configuring-a-whitelist), and don't need to sift through a mountain of convoluted materials or UI/UX from the 90s. However, they would need to add subaccount support for the diagram above to work, to read the security groups.

[^99351]: This is just my 1st impressions. [Dhruv](https://github.com/new23d) didn't pay me to write this.

### Option 4: VPC Sharing

This is a tempting, simple option, that is not well known.[^996]

[^996]: Shout out to [Aiden Steele](https://awsteele.com/blog/2022/01/02/shared-vpcs-are-underrated.html) and [Stephen Jones](https://sjramblings.io/unlock-the-hidden-power-of-vpc-sharing-in-aws) for writing their thoughts on VPC Sharing.

You can simply make a VPC in your networking account, and share private subnets to subaccounts.

This looks like:
![alt text](https://i.imgur.com/4OojfzP.png)

The main problem with this approach, is that there will _still be an Internet Gateway in the VPC_.

Unless you also used a TGW:[^12202]

![alt text](https://i.imgur.com/aYyKrH1.png)

[^12202]: Only \~$73.04 a month, plus 20 bucks per TB in data processing costs.

Assuming you don't want to pay for TGW, you can ban actions that would explicitly give an instance in a private subnet a public IP.[^220220]

[^220220]: See [the FAQ](#what-happens-if-an-ec2-instance-in-a-private-subnet-gets-a-public-ip) for what it means to have an EC2 in a private subnet have a public IP.

This can be done by banning [`ec2:RunInstances` with the `"ec2:AssociatePublicIpAddress` condition key set to `"true"`](https://github.com/ScaleSec/terraform_aws_scp/blob/521ac29d712a6ebb51feb6f11b56e6c40b61bada/security_controls_scp/modules/ec2/deny_public_ec2_ip.tf#L5-L29), and EIP related IAM actions such as `ec2:AssociateAddress`.

Then the only problem are AWS services that treat the presence of an IGW as a 'welcome mat', to make something face the Internet; [Global Accelerator](https://majinbuuonsecurity.github.io/2023/10/28/preventing-accidental-internet-exposure-of-aws-resources-part-2-handling-all-services.html#global-accelerator) is an example. These are not a big deal because so many other services don't require an IGW to make Internet facing resources, so you have to deal with the 'hundreds of AWS services' problem holistically regardless.

I discuss addressing the risk of 'hundreds of AWS services' in the [next part](https://majinbuuonsecurity.github.io/2023/10/28/preventing-accidental-internet-exposure-of-aws-resources-part-2-handling-all-services.html) of the series.

#### A Strategic Implication of VPC Sharing

In the [Shareable AWS Resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html#shareable-vpc) page of the AWS RAM documentation, `ec2:Subnet` is one of 7 resource types marked as

> Can share with ***only*** AWS accounts in its own organization.

This means you can never perform an AWS organization migration in the future.

If you are like most AWS customers:
- The Management Account of your AWS Organization has most of your resources in it
- You want to follow Best Practices[^996221] and have an empty Management Account
- It is infeasible to 'empty' out the current management account over time

[^996221]: This is Stage 1 of Scott's [AWS Security Maturity Roadmap](https://summitroute.com/downloads/aws_security_maturity_roadmap-Summit_Route.pdf), for example.

Then you will need to perform an org migration in the future, and should stay away from VPC Sharing for any environments you can't delete easily.

However, suppose you are willing to risk a production outage 😂 (Particularly if you do not use ASGs or Load Balancers, which will lose access to the subnets.) The NAT Gateway will remain functional according to AWS.

See

>Scenario 5: VPC Sharing across multiple accounts

from [Migrating accounts between AWS Organizations, a network perspective](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-accounts-between-aws-organizations-from-a-network-perspective/) by Tedy Tirtawidjaja for more information.

### Option 5: IPv6 for Egress

Remember how I said, "in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet" above. For IPv6, that's a lie.

IPv6 addresses are globally unique and, therefore, public by default. Due to this, AWS created the primitive of an [Egress-only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html), 

Unfortunately, with this primitive, there is no way to connect IPv4-only destinations, so if that is necessary -- which it likely is -- go with one of the other 3 options.

![alt text](https://i.imgur.com/wtuaa71.png)

#### More details around IPv4-only Destinations

As Sébastien Stormacq wrote in [Let Your IPv6-only Workloads Connect to IPv4 Services](https://aws.amazon.com/blogs/aws/let-your-ipv6-only-workloads-connect-to-ipv4-services/), you need only add a route table entry and set `--enable-dns64` on subnets to accomplish this -- but you unfortunately still need an IGW.

`--enable-dns64` makes it so DNS queries made to the Amazon-provided DNS Resolver will return synthetic IPv6 addresses for IPv4-only destinations with the well-known `64:ff9b::/96` prefix. The route table entry makes traffic with that prefix go to the NAT Gateway.

The problem is the NAT Gateway then needs an IGW to communicate with the destination, so one of the [other options](https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/IPv6-reference-architectures-for-AWS-and-hybrid-networks-ra.pdf) becomes necessary.

### Tradeoffs

Criteria                   | TGW                                    | PrivateLink + Egress Filtering      | GWLB + Firewall                       | VPC Sharing                           | IPv6-Only
-------------------------- | ---------------------------------------| ------------------------------------| --------------------------------------| ---------
AWS Billing Cost           | <span style="color:red">High</span>    | Low                                 | <span style="color:red">High</span>   | Low                                   | Low
Complexity*                | Medium                                 | <span style="color:red">High</span> | Medium                                | Low                                   | Medium
Scalability*               | High                                   | High                                | High                                  | Low                                   | Medium
Flexibility*               | High                                   | High                                | High                                  | Medium                                | <span style="color:red">Lowest</span>
Filtering Granularity      | None                                   | FQDN or URL Path (If desired)       | FQDN                                 | None                                 | None
Will Prevent Org Migration | False                                  | False                               | False                                 | <span style="color:red">True</span>   | False

\* = YMMV

## FAQ

### Can you walk through the cost details around Option 1?

_Note: This is assuming US East, 100 VPCs and 3 AZs. If you want to change these variables, see the [Python gist](https://gist.github.com/MajinBuuOnSecurity/361a0bac8e65432a567d5d157d2524d5) that made [the graph above](#option-1-centralized-egress-via-transit-gateway-tgw)._

#### Cost Example: No Centralized Egress

**Hourly Costs**

> For 1 NAT Gateway: $0.045 [per hour](https://aws.amazon.com/vpc/pricing/). They are also AZ-specific. So that is 3 availability zones * [730.48](https://techoverflow.net/2022/12/21/how-many-hours-are-there-in-each-month/) hours in a month * $0.045 = $98.61 per month per VPC.

> Let's say you have 100 VPCs, split across a variety of different accounts/regions etc.

> That is $9,861 a month, or $118,332 annually!

**Data Processing Costs**

> For NAT Gateway: $0.045 [per GB data processed](https://aws.amazon.com/vpc/pricing/)

> Let's say you have 100 GB a month, that is just $4.50 per month.

> 1 TB a month would be $45.

> 10 TB would be $450 a month.


#### Cost Example: Centralized Egress

**Hourly Costs**

> 1 NAT Gateway: $98.61 a month

> 1 Transit Gateway: with (100 + 1) VPC attachments, at 0.05 [per hour](https://aws.amazon.com/transit-gateway/pricing/). 101 * 730.48 * 0.05  = $3,688.92 per month.

> 9,861-(98.61+3,688.92) = A cost savings of 6,073.47 a month on hourly costs!

**Data Processing Costs**

> Same as the above + the TGW data processing charge.

> At $0.02 [per GB data processed](https://aws.amazon.com/transit-gateway/pricing/), that is only 20 bucks per TB of data!

> In conclusion, you'd need to be sending more than 303.67 TB for centralized egress to cost more.

### Why is VPC peering not a straightforward option?

The short answer is that VPC peering is not transitive, so it is not designed for you to be able to 'hop' through an IGW via it. If you change your VPC route table to send Internet-destined traffic to a VPC peering connection, the traffic won't pass through.

AWS lists this under [VPC peering limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html#vpc-peering-limitations):

> - If VPC A has an internet gateway, resources in VPC B can't use the internet gateway in VPC A to access the Internet.

> - If VPC A has a NAT device that provides Internet access to subnets in VPC A, resources in VPC B can't use the NAT device in VPC A to access the Internet.

A [longer explanation is](https://www.reddit.com/r/aws/comments/1625r2h/comment/jxxodvl):

> AWS has specific design principles and limitations for VPC peering to ensure security and network integrity. One of these limitations is that edge-to-edge routing is not supported over VPC peering connections. VPC connections are specifically designed to be non-transitive.

>This means resources in one VPC cannot access the Internet via an internet gateway or a NAT device in a peer VPC. AWS does not propagate packets destined for the Internet from one VPC to another over a peering connection, even if you try configuring NAT at the instance level.

>The primary reason for this limitation is to maintain a clear network boundary and enforce security policies. If AWS allowed traffic from VPC B to Egress to the Internet through VPC A's NAT gateway, it would essentially make VPC A a transit VPC, which breaks the AWS design principle of VPC peering as a non-transitive relationship.

### Why is VPC PrivateLink not a straightforward option?

The reasoning is similar to peering: PrivateLink is not meant to be transitive.

To dive deeper: When you make an interface VPC endpoint with AWS PrivateLink, a "[requester-managed network interface](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/requester-managed-eni.html)" is created. The "requester" is AWS, as you can see by the mysterious "727180483921" account ID.

If you try to disable "[Source/destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics)" (which ensures that the ENI is either the source or the destination of any traffic it receives), you will not be able to, since it is managed by AWS. So traffic is dropped before it would ever travel cross-account.

### How much cheaper is VPC Sharing than TGW?

_Assuming you would have 50 spoke VPCs and were in US East._

If you are making 1 giant VPC, and sharing different subnets to each subaccount, then both options would have 1 NAT Gateway. The only difference is a TGW:

> 1 Transit Gateway: with (50 + 1) VPC attachments, at 0.05 [per hour](https://aws.amazon.com/transit-gateway/pricing/). 51 * 730.48 * 0.05  = $1,862.72 per month.

> At $0.02 [per GB data processed](https://aws.amazon.com/transit-gateway/pricing/), that is 20 bucks a month more per TB of data!

Assuming 1 TB of data processed a month, that is $1,882.72 more a month.

#### VPC Endpoint Costs

If you wanted to use VPC endpoints across all VPCs, then you'd have to pay 50x more for them with TGW.

Those are $0.01 per hour per availability zone = $22 per month per service per VPC. So for e.g. `SQS, SNS, KMS, STS, X-Ray, ECR, ECS` across all VPCs that is $154 a month with sharing.

Vs $7,700 a month with 50 separate VPCs!

### How do I access my machines if they are all in private subnets?

Use [SSM](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html), or a similar product.

### What happens if an EC2 instance in a private subnet gets a public IP?

You can send packets to it from the Internet. However, the EC2 can't really respond.

This is because incoming traffic first hits the IGW, then the EC2. Nothing else is checked assuming the NACL and security group allow it.

As for why it cannot respond to traffic. For a private subnet, the route table -- which is only consulted for outgoing traffic -- will have a path to a NAT Gateway, not the IGW. So response packets will have the source IP of the NAT Gateway rather than the EC2 instance, messing up e.g. the TCP handshake.[^9133]

[^9133]: A 3rd shout out to Aiden Steele who [wrote about this in another context](https://twitter.com/__steele/status/1572752577648726016), and has [visuals / code here](https://github.com/aidansteele/matconnect#matconnect).

With VPC sharing, you can control the NACL but not the Security Group. A NACL won't be able to help however, because an Ingress deny rule blocking the Internet from hitting the EC2 will also block responses from the Internet to Egress Traffic.

## Open Questions

- Are there no VPC flow logs for traffic that gets dropped due to a source/destination check on an ENI?
- If there are no VPC flow logs for traffic, there won't be any possibility of mirroring that traffic either, right?
- Are there any other options for Egress that I missed?
- Did I get anything wrong? I'm sure the Internet will let me know in spades.

## Conclusion

Let me know how it goes limiting your Internet-exposed attack surface in an easy to understand, secure-by-default way. 

You might still get breached, but hopefully in a more interesting way.

![alt text](https://media.tenor.com/YrcU9HOzqmUAAAAC/majin-buu-clap.gif)

The [next part](https://majinbuuonsecurity.github.io/2023/10/28/preventing-accidental-internet-exposure-of-aws-resources-part-2-handling-all-services.html) of this series covers handling the hundreds of other AWS services beyond just those in a VPC.

## Footnotes
