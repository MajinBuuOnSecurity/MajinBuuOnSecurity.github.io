---
layout: post
toc: true
title: "Preventing Accidental Internet-Exposure of AWS Resources"
---

Many AWS customers have suffered breaches due to exposing resources to the Internet by accident. [^1] This post walks through the different ways to mitigate that risk.

[^1]: See [this post](https://maia.crimew.gay/posts/how-to-hack-an-airline/) for an example breach, one of many [AWS customer security incidents](https://github.com/ramimac/aws-customer-security-incidents#background).


## What Good Looks Like

S3 is far simpler to secure than resources in a VPC, so let us start there to get a sense of what we are aiming for.

In your multi-account AWS strategy, most accounts should have [account-wide public block access for S3](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_account_public_access_block) enabled at account creation time and have that control made immutable [via SCP](https://summitroute.com/blog/2020/03/25/aws_scp_best_practices/#protect-security-settings). This eliminates the possibility of S3 public data leaks. 

When a public S3 bucket is needed, it should be made in a special `S3 Public Resources` account (under a separate OU) where only public S3 buckets can live.[^2]

[^2]: New S3 buckets are private by default now, but being able to look at your AWS Org structure from a thousand-foot view and know which subtree can have public S3 buckets is invaluable.

A simplified AWS organization structure supporting this is:

![alt text](https://i.imgur.com/bPIKZoC.png)

## About The Problem

The question this post answers is: How do you implement the same strategy for resources in a VPC (EC2 instances, ELBs, RDS databases, etc.)?


This is more complicated, because in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

The reason they are tightly coupled: is an Internet Gateway (IGW) is necessary for both. So if you have IAM permissions to make an IGW, or are unrestrained in how you create resources in a VPC that has one, you can expose resources to the Internet.

There are many ways to make this mistake, but the following options limit almost all of them. [^777]

[^777]: Read the full post, particularly the [Stopping The Bleeding](http://localhost:4000/2023/09/13/preventing-accidental-internet-exposure-of-aws-resources.html#stopping-the-bleeding) section, for more in-depth look.

## Preventing by Design

The solution is to architect your network so subaccount owners can only use IGWs for Egress. This is done by what is called [Centralized Egress](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/centralized-egress.html), and there are a few ways to implement it.

This will also save you NATGW money, which is a win for your partner teams. (TODO: Word this about persuasion.)

### IPv6 for Egress

Remember how I said "in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet" above. For IPv6, that's a lie.

IPv6 addresses are globally unique, and therefore public by default. Due to this, AWS created the primitive of an [Egress-only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html), 

With this primitive, you are off to the races.


### Transit Gateway (TGW)

This is the most common way of implementing this, and probably the best. If money is no issue for you: go this route.




#### A Note On Cost

On one hand, AWS states:

>Deploying a NAT gateway in every spoke VPC can become cost prohibitive because you pay an hourly charge for every NAT gateway you deploy (refer to Amazon VPC pricing), so centralizing it could be a viable option. 

On the other hand, 

>In some edge cases when you send huge amounts of data through NAT gateway from a VPC, keeping the NAT local in the VPC to avoid the Transit Gateway data processing charge might be a more cost-effective option.

Sending huge amounts of data through a NAT Gateway should be avoided anyway.
S3, Splunk, and [Honeycomb](https://docs.honeycomb.io/integrations/aws/aws-privatelink/) and similar companies have VPC endpoints. 


### VPC Peering with Egress Filtering

VPC Peering is mostly not an option. See the [FAQ](http://localhost:4000/2023/09/13/preventing-accidental-internet-exposure-of-aws-resources.html#why-is-vpc-peering-not-an-option) for more information.

However, if you are will to do a lot of heavy lifting that is orthogonal to AWS primitives, you _can_ use peering in combination with [Internet Egress Filtering](https://eng.lyft.com/internet-egress-filtering-of-services-at-lyft-72e99e29a4d9), to accomplish centralized egress. This is because the destination IP of outbound traffic won't be the Internet, but the private IP of the proxy living in the peered VPC. Therefore, it will happily pass through the peering connection. This is assuming you are deploying e.g. iptables to re-route Internet-destined traffic on every host.

Some reasons you may not want to do this are:
- Significant effort
- It won't be possible to do for all subaccount types, such as test/sandbox accounts. (Where requiring `iptables` and proxies is too heavy weight.)
- Egress filtering (P2/P3) is further down the security maturity roadmap than preventing accidental Internet-exposure (P1). So coupling the two may not make strategic sense.
- If something goes wrong on the host, the lost traffic will not appear in VPC flow logs [^98] or traffic mirroring logs.[^985] The DNS lookups will show up in Route53 query logs, but that's it.

With that said, AWS does not have a primitive to perform Egress filtering [^99], and so you will have to implement Egress filtering via a proxy eventually. Therefore, in production accounts you may choose to go `peering -> proxy`. And for sandbox accounts, use a different centralized egress pattern e.g. VPC sharing (which will not disrupt an org migration due to their ephemeral nature).


[^98]: [Flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-limitations) does not state, "Internet-bound traffic sent to a peering connection" under `The following types of traffic are not logged:`. After testing I believe it should.

[^985]: A peering connection cannot be selected as a [traffic mirror target](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-targets.html).

[^99]: It has AWS Network Firewall, which can be fooled via SNI spoofing. So it is at best a stepping stone to keep and inventory of your Egress traffic if you can't get a proxy up and running short-term.


### VPC Sharing

Simple, and not well known.

#### A Strategic Implication of VPC Sharing

In the [Shareable AWS Resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html#shareable-vpc) page of the AWS RAM documentation.

It states `ec2:Subnet` is one of 7 resources types that is marked as

> Can share with ***only*** AWS accounts in its own organization.

At first glance, this appears to mean you cannot perform an AWS organization migration ever in the future.

However, if you are willing to risk a production outage 😂 (Particularly if you do not use ASGs or Load Balancers, which will lose access to the subnets), the NAT Gateway will remain functional according to AWS.

See

>Scenario 5: VPC Sharing across multiple accounts

from [Migrating accounts between AWS Organizations from a network perspective](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-accounts-between-aws-organizations-from-a-network-perspective/) for more information.

### Tradeoffs

You almost certainly do not need a table.

Go with TGW, if you can't do VPC Sharing.

IPv6-only probably won't fly at your organization. But you know best.


Criteria                   | TGW                   | VPC Sharing           | IPv6-Only
-------------------------- | --------------------- | --------------------- | ---------------------
AWS Billing Cost           | Highest               | Low                   | Low
Complexity*                | Low                   | Low                   | Medium
Scalability*               | High                  | Low                   | High
Flexibility*               | Highest               | Medium                | Low
Will Prevent Org Migration | False                 | True                  | False

\* = YMMV

## Stopping The Bleeding

What if you have a giant monolithic account with a mix of private and public assets, that you didn't apply these design principles to?

All hope is not lost, but you and I are going to need to play whack-a-mole together.

![alt text](https://media.tenor.com/hGclJ34JeSIAAAAC/one-punch.gif)

### How It Happens


1. A VPC gets an Internet Gateway
2. A public-facing resource gets created
2a. If it is an EC2, the instance needs to be in a public subnet. Or, if it was via run-instances, be given --associate-public-ip-address
3. (Optional) If the resource in 2 was an ELB, a lambda / IP / EC2 instance can be added as its targets.
4. (Optional) If the resource in 2 was an EIP, an ENI or EC2 instance can be associated with it.

If you don’t provision accounts properly, they will all have a default VPC in every region, with every subnet public.




## Short comings

- `eks:CreateCluster`
- New IAM Actions

For new IAM Services, you should have an allowlist strategy that limits the amount of AWS services you need to know in-depth.
For Sandbox accounts, this is not feasible. 

- aws-nuke + the Cloud Control API (Or actually nuking accounts, which is more interesting.)
- Written engineering standards about only having public data (Part of a larger Data Classication Policy.)

## FAQ

### Why is VPC peering not a straightforward option?

The short-answer is that, VPC peering is not transitive, so it is not designed for you to be able to 'hop' through an IGW via it. If you change your VPC route table to send Internet-destined traffic to a VPC peering connection, the traffic won't pass through it.

AWS lists this under [VPC peering limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html#vpc-peering-limitations):

> - If VPC A has an internet gateway, resources in VPC B can't use the internet gateway in VPC A to access the internet.

> - If VPC A has an NAT device that provides internet access to subnets in VPC A, resources in VPC B can't use the NAT device in VPC A to access the internet.

A [longer explanation is](https://www.reddit.com/r/aws/comments/1625r2h/comment/jxxodvl):

> AWS has specific design principles and limitations in place for VPC peering to ensure security and network integrity. One of these limitations is that edge-to-edge routing is not supported over VPC peering connections. VPC connections are specifically designed to be non-transitive.

>This means resources in one VPC cannot use an internet gateway or a NAT device in a peer VPC to access the internet. AWS does not propagate packets that are destined for the internet from one VPC to another over a peering connection, even if you try to configure NAT at the instance level.

>The primary reason for this limitation is to maintain a clear network boundary and enforce security policies. If AWS allowed traffic from VPC B to egress to the internet through VPC A's NAT gateway, it would essentially make VPC A a transit VPC, which breaks the AWS design principle of VPC peering as a non-transitive relationship.

### How do I access my machines if not public IP & IP whitelist, or DirectConnect->PrivateLink?

Use SSM.



## Seeking Feedback

- Are there any other options?
- What are the strategic implications that I missed?
- How can I demonstrate depth here?

## Conclusion

Once you have centralized your Egress traffic, and eliminated the high-priority risk of Internet-exposure. It is a good idea to think about filtering that traffic.

At each spoke: DNS via AWS DNS Firewall
In the center: via Envoy or Smokescreen.

It is the next step in your security maturity roadmap.

## Appendix A: AWS Networking Basics


- NAT Gateway
- Internet Gateway
- Elastic Load Balancers


