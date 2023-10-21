---
layout: post
toc: true
title: "Preventing Accidental Internet-Exposure of AWS Resources"
---

Many AWS customers have suffered breaches due to exposing resources to the Internet by accident. [^1] This post walks through the different ways to mitigate that risk.

[^1]: See [this post](https://maia.crimew.gay/posts/how-to-hack-an-airline/) for an example breach, one of many [AWS customer security incidents](https://github.com/ramimac/aws-customer-security-incidents#background).


## What Good Looks Like

S3 is far simpler to secure than resources in a VPC, so let's use that as an example to get a sense of what we are aiming for.

In your multi-account AWS strategy, most accounts should have [account-wide public block access for S3](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_account_public_access_block) enabled at account creation time and have that control made immutable [via SCP](https://summitroute.com/blog/2020/03/25/aws_scp_best_practices/#protect-security-settings), eliminating the possibility of S3 public data leaks. 

When a public S3 bucket is needed, it should be made in a special `S3 Public Resources` account (under a separate OU) where only public S3 buckets can live.[^2]

[^2]: New S3 buckets are private by default now, but being able to look at your AWS Org structure from a thousand-foot view and know which subtree can have public S3 buckets is invaluable.

A simplified AWS organization structure supporting this is:

![alt text](https://i.imgur.com/bPIKZoC.png)

## About The Problem

The question this post answers is: How do you implement the same strategy for resources in a VPC (EC2 instances, ELBs, RDS databases, etc.)?


This is more complicated, because in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

The reason they are tightly coupled: is an Internet Gateway (IGW) is necessary for both. So if an engineer has IAM permissions to create an IGW, or is unrestrained in how they create resources in a VPC that has one, they can expose resources to the Internet.

The Egress use-case typical looks like:
![alt text](https://i.imgur.com/vKsdNOh.png)

Whereas an accidental Internet exposure might look like:

![alt text](https://i.imgur.com/utMw64L.png) 

Or:

![alt text](https://i.imgur.com/i65kHne.png)

There are many ways to expose resources to the Internet like this, but the key insight is that they all require an Internet Gateway (IGW). [^777]

[^777]: Read the full post, particularly the [Stopping The Bleeding](#stopping-the-bleeding) section, for more in-depth look.

## Preventing by Design

Since an IGW is the root of all of our problems, we simply can ban their creation via SCP 
(`"ec2:CreateInternetGateway"`) upon vending an account, (alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.)

That's it. Problem solved! You can hand the subaccount over to the customer, they will never be able to make public-facing load balancers or EC2 instances regardless of their IAM permissions.

You can then, put accounts that were vended this way with the IGW-deny-SCP, in an OU and achieve [What Good Looks Like](#what-good-looks-like) above.

But what if you need to support the Egress use-case above? Then you have 4 good options.

### Option 1: Centralized Egress via Transit Gateway (TGW)

This is the most common way of implementing this, and probably the best. If money is no issue for you: go this route.

It looks like this:

![alt text](https://i.imgur.com/alRH2hN.png) 

AWS first wrote about this [in 2019](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-a-single-internet-exit-point-from-multiple-vpcs-using-aws-transit-gateway/)[^5555] and lists it under their prescriptive guidance as [Centralized Egress](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/centralized-egress.html).

As you can see, each VPC in a subaccount has a route table that has 0.0.0.0/0 destined traffic sent to a TGW in another account, where an IGW does live.

[^5555]: Let me know if you find an earlier reference.

#### A Note On Cost

This will likely save you money on NAT Gateways, which are one of -- if not thee -- most expensive networking components in AWS.

On one hand, [AWS states](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-egress-to-internet.html):

>Deploying a NAT gateway in every spoke VPC can become cost prohibitive because you pay an hourly charge for every NAT gateway you deploy (refer to Amazon VPC pricing), so centralizing it could be a viable option. 

On the other hand, they also say:

>In some edge cases when you send huge amounts of data through NAT gateway from a VPC, keeping the NAT local in the VPC to avoid the Transit Gateway data processing charge might be a more cost-effective option.

Sending huge amounts of data through a NAT Gateway should be avoided anyway.
[S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html), [Splunk](https://www.splunk.com/en_us/blog/platform/announcing-aws-privatelink-support-on-splunk-cloud-platform.html), [Honeycomb](https://docs.honeycomb.io/integrations/aws/aws-privatelink/) and similar companies have VPC endpoints you can utilize to lower NAT Gateway data processing charges. 


### Option 2: IPv6 for Egress

Remember how I said "in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet" above. For IPv6, that's a lie.

IPv6 addresses are globally unique, and therefore public by default. Due to this, AWS created the primitive of an [Egress-only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html), 

With this primitive, you are off to the races.


### Option 3: Centralized Egress via PrivateLink / VPC Peering with Egress Filtering

VPC Peering or PrivateLink are mostly non-options. See the [FAQ](#why-is-vpc-peering-not-a-straightforward-option) for more information.

However, if you are will to do a lot of heavy lifting that is orthogonal to AWS primitives, you _can_ use these in combination with [Internet Egress Filtering](https://eng.lyft.com/internet-egress-filtering-of-services-at-lyft-72e99e29a4d9), to accomplish centralized egress. This is because the destination IP of outbound traffic won't be the Internet, but a private IP.

This is assuming you are deploying e.g. iptables to re-route Internet-destined traffic on every host.

Some reasons you may not want to do this are:
- Significant effort
- It won't be possible to do for all subaccount types, such as test/sandbox accounts. (Where requiring `iptables` and a proxy is too heavy weight.)
- Egress filtering (P2/P3) is further down the security maturity roadmap than preventing accidental Internet-exposure (P1). So tightly coupling the two, and needing to setup a proxy first, may not make strategic sense.
- If something goes wrong on the host, the lost traffic will not appear in VPC flow logs [^98] or traffic mirroring logs.[^985] The DNS lookups will show up in Route53 query logs, but that's it.

With that said, AWS does not have a primitive to perform Egress filtering [^99], and so you will have to implement Egress filtering via a proxy eventually. Therefore, in production accounts you may choose to go with this option. And for sandbox accounts, use a different centralized egress pattern e.g. VPC sharing (which will not disrupt an org migration due to their ephemeral nature).

Using PrivateLink, in this way, would look like this:

![alt text](https://i.imgur.com/5Vh2SuX.png)

[^98]: [Flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-limitations) does not state, "Internet-bound traffic sent to a peering connection" or "Internet-bound traffic sent to a VPC interface endpoint." under `The following types of traffic are not logged:`. After testing I believe it should, but these are likely omitted due to not being a proper use-case.

[^985]: A peering connection cannot be selected as a [traffic mirror source or target](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-targets.html), but a network interface can. However, only an ENI belonging to an EC2 instance can be a mirror source, not an ENI belonging to an Interface endpoint. The documentation doesn't mention this anywhere I could find.

[^99]: It has AWS Network Firewall, which can be fooled via SNI spoofing. So it is at best a stepping stone to keep an inventory of your Egress traffic if you can't get a proxy up and running short-term.


### Option 4: VPC Sharing

This is the simplest option, and not well known.[^996]

You can simply make a VPC in your networking account, and share private subnets to subaccounts.

![alt text](https://i.imgur.com/4OojfzP.png)

[^996]: Shout out to [Aiden Steele](https://awsteele.com/blog/2022/01/02/shared-vpcs-are-underrated.html) and [Stephen Jones](https://sjramblings.io/unlock-the-hidden-power-of-vpc-sharing-in-aws) for writing their thoughts on VPC Sharing.


#### A Strategic Implication of VPC Sharing

In the [Shareable AWS Resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html#shareable-vpc) page of the AWS RAM documentation, `ec2:Subnet` is one of 7 resources types marked as

> Can share with ***only*** AWS accounts in its own organization.

This means you can never perform an AWS organization migration in the future, so if you are 99% of AWS customers, do not use VPC sharing.

However, if you are willing to risk a production outage 😂 (Particularly if you do not use ASGs or Load Balancers, which will lose access to the subnets), the NAT Gateway will remain functional according to AWS.

See

>Scenario 5: VPC Sharing across multiple accounts

from [Migrating accounts between AWS Organizations from a network perspective](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-accounts-between-aws-organizations-from-a-network-perspective/) by 
Tedy Tirtawidjaja for more information.


### Tradeoffs

My recommendation: Go with TGW, if you can't do VPC Sharing. Once you are ready to implement Egress Filtering, have  assets use PrivateLink + Egress Filtering.

IPv6-only probably won't fly at your organization. But you know best.

The important thing is, no matter which direction your networking team wants to go in, this is doable.

Criteria                   | TGW                   | VPC Sharing           | IPv6-Only             | PrivateLink + Egress Filtering
-------------------------- | --------------------- | --------------------- | --------------------- | ---------------------
AWS Billing Cost           | <span style="color:red">Highest</span> | Lowest                              | Low                                | Medium
Complexity*                | Low                                    | Low                                 | Medium                             | <span style="color:red">High</span>
Scalability*               | High                                   | Low                                 | High                               | High
Flexibility*               | High                                   | Medium                              | <span style="color:red">Low</span> | High
Will Prevent Org Migration | False                                  | <span style="color:red">True</span> | False                              | False

\* = YMMV

## Stopping The Bleeding

What if you have a giant monolithic account with a mix of private and public assets, that you didn't apply these design principles to?

All hope is not lost, but you and I are going to need to play whack-a-mole together.

![alt text](https://media.tenor.com/hGclJ34JeSIAAAAC/one-punch.gif)

### How It Happens

![alt text](https://i.imgur.com/gyXZz2E.gif)

1. A VPC gets an Internet Gateway
2. A public-facing resource gets created
	- If it is an EC2, the instance needs to be in a public subnet. Or, if it was created via `ec2:RunInstances`, be given the `--associate-public-ip-address` option.
3. (Optional) If the resource in 2 was an ELB, a lambda / IP / EC2 instance can be added as its targets.
4. (Optional) If the resource in 2 was an EIP, an ENI or EC2 instance can be associated with it.

### How To Stop It

Banning IAM Actions
- Cannot create Load Balancers
- Cannot create EIPs
- Cannot modify target groups
- Cannot associate EIPs

Banning IAM Actions Conditionally
- Cannot touch public subnets
- Cannot `ec2:RunInstances` with 
- Cannot modify target groups of public load balancers



## Short comings

- `eks:CreateCluster`
- `globalaccelerator:Create*`
- New IAM Actions

For new IAM Services, you should have an allowlist strategy that limits the amount of AWS services you need to know in-depth.

For Sandbox accounts, this is not feasible, so you will need to manually maintain a deny list of these IAM Actions. 

(Granted, you should be re-creating sandbox accounts [or less ideally, nuking] regularly and only have public data in them.)

## FAQ

### Why is VPC peering not a straightforward option?

The short-answer is that, VPC peering is not transitive, so it is not designed for you to be able to 'hop' through an IGW via it. If you change your VPC route table to send Internet-destined traffic to a VPC peering connection, the traffic won't pass through it.

AWS lists this under [VPC peering limitations](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html#vpc-peering-limitations):

> - If VPC A has an internet gateway, resources in VPC B can't use the internet gateway in VPC A to access the Internet.

> - If VPC A has an NAT device that provides Internet access to subnets in VPC A, resources in VPC B can't use the NAT device in VPC A to access the Internet.

A [longer explanation is](https://www.reddit.com/r/aws/comments/1625r2h/comment/jxxodvl):

> AWS has specific design principles and limitations in place for VPC peering to ensure security and network integrity. One of these limitations is that edge-to-edge routing is not supported over VPC peering connections. VPC connections are specifically designed to be non-transitive.

>This means resources in one VPC cannot use an internet gateway or a NAT device in a peer VPC to access the Internet. AWS does not propagate packets that are destined for the Internet from one VPC to another over a peering connection, even if you try to configure NAT at the instance level.

>The primary reason for this limitation is to maintain a clear network boundary and enforce security policies. If AWS allowed traffic from VPC B to egress to the Internet through VPC A's NAT gateway, it would essentially make VPC A a transit VPC, which breaks the AWS design principle of VPC peering as a non-transitive relationship.

### Why is VPC PrivateLink not a straightforward option?

The reasoning is similar to peering: PrivateLink is not meant to be transitive.

To dive deeper: When you make an interface VPC endpoint with AWS PrivateLink, a "[requester-managed network interface](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/requester-managed-eni.html)" is created. The "requester" is AWS, as you can see by the mysterious "727180483921" account ID.

If you try to disable "[Source/destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics)", (which ensures that the ENI is either the source or the destination of any traffic it receives), you will not be able to. So traffic is dropped before it would ever travel cross-account.

### How do I access my machines if they are all in private subnets?

Use [SSM](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html).


## Open Questions

- Are there no VPC flow logs for traffic that gets dropped due to a source/destination check on an ENI?
- If there are no VPC flow logs for traffic, there won't be any possibility of mirroring that traffic either, right?
- Are there any other options for Egress that I missed?

## Conclusion

Let me know how it goes limiting your Internet-exposed attack surface in an easy to understand, secure-by-default way. 

You might still get breached, but hopefully in a more interesting way.


![alt text](https://media.tenor.com/YrcU9HOzqmUAAAAC/majin-buu-clap.gif)

## Footnotes

   

