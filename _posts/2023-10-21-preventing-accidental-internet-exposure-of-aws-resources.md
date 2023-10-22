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

[^2]: [New S3 buckets have Public Access Block by default now](https://aws.amazon.com/about-aws/whats-new/2022/12/amazon-s3-automatically-enable-block-public-access-disable-access-control-lists-buckets-april-2023/), but being able to look at your AWS Org structure from a thousand-foot view and know which subtree can have public S3 buckets is invaluable.

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

There are many ways to expose resources to the Internet like this, but the key insight is that they all require an Internet Gateway (IGW).

## Preventing by Design

Since an IGW is the root of all of our problems, we simply can ban their creation via SCP 
(`"ec2:CreateInternetGateway"`) upon vending an account, (alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.)

That's it. Problem solved! You can hand the subaccount over to the customer, they will never be able to make public-facing load balancers or EC2 instances regardless of their IAM permissions.

You can then, put accounts that were vended this way with the IGW-deny-SCP, in an OU and achieve [What Good Looks Like](#what-good-looks-like) above.

But what if you need to support the Egress use-case above? Then you have 4 possible options.

### Option 1: Centralized Egress via Transit Gateway (TGW)

This is the most common implemention, and probably the best. If money is no issue for you: go this route.

It looks like this:

![alt text](https://i.imgur.com/alRH2hN.png) 

AWS first wrote about this [in 2019](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-a-single-internet-exit-point-from-multiple-vpcs-using-aws-transit-gateway/)[^5555] and lists it under their prescriptive guidance as [Centralized Egress](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/centralized-egress.html).

As you can see, each VPC in a subaccount has a route table that has 0.0.0.0/0 destined traffic sent to a TGW in another account, where an IGW does live.

[^5555]: Let me know if you find an earlier reference.

#### A Note On Cost

This will save you money on NAT Gateways, which is arguably the most notoriously expensive networking component in AWS.

On one hand, [AWS states](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-egress-to-internet.html):

>Deploying a NAT gateway in every AZ of every spoke VPC can become cost prohibitive because you pay an hourly charge for every NAT gateway you deploy (refer to Amazon VPC pricing), so centralizing it could be a viable option. 

On the other hand, they also say:

>In some edge cases when you send huge amounts of data through NAT gateway from a VPC, keeping the NAT local in the VPC to avoid the Transit Gateway data processing charge might be a more cost-effective option.

Sending huge amounts of data through a NAT Gateway should be avoided anyway.
[S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html), [Splunk](https://www.splunk.com/en_us/blog/platform/announcing-aws-privatelink-support-on-splunk-cloud-platform.html), [Honeycomb](https://docs.honeycomb.io/integrations/aws/aws-privatelink/) and similar companies have VPC endpoints you can utilize to lower NAT Gateway data processing charges.

### Option 2: Centralized Egress via PrivateLink / VPC Peering with Egress Filtering

VPC Peering or PrivateLink are mostly non-options. See the [FAQ](#why-is-vpc-peering-not-a-straightforward-option) for more information.

However, if you are will to do a lot of heavy lifting that is orthogonal to AWS primitives, you _can_ use these in combination with [Internet Egress Filtering](https://eng.lyft.com/internet-egress-filtering-of-services-at-lyft-72e99e29a4d9), to accomplish centralized egress. This is because the destination IP of outbound traffic won't be the Internet, but a private IP.

This is assuming you are deploying e.g. iptables to re-route Internet-destined traffic on every host.

Some reasons you may not want to do this are:
- Significant effort
- It won't be possible to do for all subaccount types, such as sandbox accounts. (Where requiring `iptables` and a proxy are too heavy weight.)
- Egress filtering (P2/P3) is further down the security maturity roadmap than preventing accidental Internet-exposure (P1). So tightly coupling the two, and needing to setup a proxy first, may not make strategic sense.
- If something goes wrong on the host, the lost traffic will not appear in VPC flow logs [^98] or traffic mirroring logs.[^985] The DNS lookups will show up in Route53 query logs, but that's it.

With that said, AWS does not have a primitive to perform Egress filtering [^99], and so you will have to implement Egress filtering via a proxy eventually. Therefore, in production accounts you may choose to go with this option. And for sandbox accounts, use a different centralized egress pattern e.g. VPC sharing (which will not disrupt an org migration due to their ephemeral nature).

Using PrivateLink, in this way, would look like this:

![alt text](https://i.imgur.com/5Vh2SuX.png)

[^98]: [Flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-limitations) does not state, "Internet-bound traffic sent to a peering connection" or "Internet-bound traffic sent to a VPC interface endpoint." under `The following types of traffic are not logged:`. After testing I believe it should, but these are likely omitted due to not being a proper use-case.

[^985]: A peering connection cannot be selected as a [traffic mirror source or target](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-targets.html), but a network interface can. However, only an ENI belonging to an EC2 instance can be a mirror source, not an ENI belonging to an Interface endpoint. The documentation doesn't mention this anywhere I could find.

[^99]: It has [AWS Network Firewall](https://aws.amazon.com/network-firewall/faqs/), which can be fooled via SNI spoofing. So it is at best a stepping stone to keep an inventory of your Egress traffic if you can't get a proxy up and running short-term, and are not using TLS 1.3 with encrypted client hello (ECH) or encrypted SNI (ESNI)..


### Option 3: VPC Sharing

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

### Option 4: IPv6 for Egress

Remember how I said "in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet" above. For IPv6, that's a lie.

IPv6 addresses are globally unique, and therefore public by default. Due to this, AWS created the primitive of an [Egress-only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html), 

Unfortunately, with this primitive, there is no way to connect IPv4-only destinations, so if that is necessary, go with one of the other 3 options.

![alt text](https://i.imgur.com/wtuaa71.png)

#### More details around IPv4-only Destinations

As Sébastien Stormacq wrote in [Let Your IPv6-only Workloads Connect to IPv4 Services](https://aws.amazon.com/blogs/aws/let-your-ipv6-only-workloads-connect-to-ipv4-services/), you need only add a route table entry and set `--enable-dns64` on subnets to accomplish this -- but you unfortunately still need an IGW.

DNS queries made to the Amazon-provided DNS Resolver in subnets will then return synthetic IPv6 addresses for IPv4-only destinations with the well-known `64:ff9b::/96` prefix. And due to the route table entry, traffic with that prefix will be sent to the NAT Gateway.

The problem is the NAT Gateway then needs an IGW to communicate with the destination, so one of the [other options](https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/IPv6-reference-architectures-for-AWS-and-hybrid-networks-ra.pdf) becomes necessary.

### Tradeoffs

My recommendation: Go with TGW, if you can't do VPC Sharing. Once you are ready to implement Egress Filtering, have  assets use PrivateLink + Egress Filtering.

Being limited to IPv6-only destinations is likely unfeasible for your current or future use-cases, but you know best.

The important thing is, no matter which direction your networking team wants to go in, you can ban IGWs.

Criteria                   | TGW                   | VPC Sharing           | IPv6-Only             | PrivateLink + Egress Filtering
-------------------------- | --------------------- | --------------------- | --------------------- | ---------------------
AWS Billing Cost           | <span style="color:red">Highest</span> | Lowest                              | Low                                | Medium
Complexity*                | Medium                                 | Low                                 | Medium                             | <span style="color:red">High</span>
Scalability*               | High                                   | Low                                 | High                               | High
Flexibility*               | High                                   | Medium                              | <span style="color:red">Lowest</span> | High
Will Prevent Org Migration | False                                  | <span style="color:red">True</span> | False                              | False

\* = YMMV

## Stopping The Bleeding

What if you have a giant monolithic account with a mix of private and public assets, that you didn't apply these design principles to?

All hope is not lost, but you and I are going to need to play whack-a-mole together.

![alt text](https://media.tenor.com/hGclJ34JeSIAAAAC/one-punch.gif)

### IAM Protection Rings

Think about your IAM roles as [privilege rings](https://en.wikipedia.org/wiki/Protection_ring) in x86:

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/1350px-Priv_rings.svg.png" alt="drawing" width="500"/>

In AWS, rings 1 through 3 can be different types of IAM roles:

- Ring 0 is SCPs
- Ring 1 can make public facing resources
- Ring 2 needs to e.g. create load balancers, but none of them should be Internet-facing.
- Ring 3 needs to e.g. create EC2 instances, but none of them should be Internet-facing.

With an SCP banning `ec2:CreateInternetGateway`, rings 1 through 3 cannot create Internet-facing assets no matter what, as ring 0 prevents them -- but without this, we need to play IAM games.

#### Supporting Ring 3 with RunInstances

For Ring 3, we should be able to allow the role to call `ec2:RunInstances`, under the condition that [`ec2:AssociatePublicIpAddress`](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html#amazonec2-policy-keys) is `false`. 

Except that is insufficent: if the [subnet given](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/run-instances.html) has [`"map-public-ip-on-launch"`](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/modify-subnet-attribute.html#options) set to true, the EC2 will get a public IP.

So we need to allowlist the subnets that have that attribute set to false, by adding the condition that [`ec2:SubnetID`](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html#amazonec2-policy-keys) equals an ID on the allowlist.

Maintaining such a list of opaque IDs would be tedious, so in Terraform we can use a data source to grab them:
```go
data "aws_subnets" "public" {
 filter {
   name = "map-public-ip-on-launch"
   values = [false]
 }
}
```

Except that is insufficient: as it does not read from every region.[^1404]

So we need to write [our own custom Terraform provider](https://gist.github.com/MajinBuuOnSecurity/cb6b4689b47f555a2324c3f33da8e7eb#file-public_subnets-go-L172-L181) to iterate through all regions.

[^1404]: Even if it did, [`"aws_subnets"`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/subnets) is one of the [better](https://github.com/hashicorp/terraform/issues/16380#issuecomment-418476841) data sources. [aws_lb](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/lb) and [aws_lbs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/lbs) do not have filters, and the latter fails if there are none.

Finally we can create an [`aws_iam_policy_document`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document)[^1427] similar to the following abbreviated[^1428] policy:

[^1427]: See [this gist](https://gist.github.com/MajinBuuOnSecurity/205273d308435f8d4a52759836aeb9e5) for a denylist example.
[^1428]: If going with an allowlist approach, you also need statements that cover all other resource types (image, instance, security-group, volume etc.) for these actions. 


```go
data "private_subnets" "subnets" {
  regions = var.regions_in_use
}

data "aws_iam_policy_document" "allow_private_subnets" {
  statement {
    sid = "AllowSubnetConds"

    actions = [
      "ec2:RunInstances",
    ]

    effect = "Allow"

    resources = [
      "arn:aws:ec2:*:*:subnet/*"
    ]

    condition {
      test     = "StringEquals"
      variable = "ec2:SubnetID"

      values = data.private_subnets.subnets.ids
    }
  }

  statement {
    sid = "AllowENIConds"

    actions = [
      "ec2:RunInstances",
    ]

    effect = "Allow"

    resources = [
      "arn:aws:ec2:*:*:network-interface/*",
    ]

    condition {
      test     = "Bool"
      variable = "ec2:AssociatePublicIpAddress"

      values = [
        "false",
      ]
    }

    condition {
      test     = "StringEquals"
      variable = "ec2:Subnet"

      values = data.private_subnets.subnets.arns
    }
  }
}
```

Now when someone in Ring 1 or 2 creates a new subnet, they can apply this dynamic policy as well so Ring 3 can automatically launch instances in it.

This is  <font size="+2"><strong>one mole</strong></font> we just whacked. There are many others. This is What Bad Looks Like.

### How It Happens

Let's see if we can list every possible mole that can pop up.

Creation of load balancer:

...TODO...

Creation of instance in public subnet:

...TODO...

Target group of load balancer

![alt text](https://i.imgur.com/gyXZz2E.gif)

Lambda gets put as target

...TODO...

IP gets put as target

...TODO...

EIP gets associated with ENI

...TODO...

EIP gets associated with EC2

...TODO...


TODO: Can I edit the scheme of a load balancer? It doesn't look like it, but I'll try.

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

While it is true in almost all cases, that an Internet Gateway (IGW) is a prerequisite to expose an asset to the Internet, there is one exception I am aware of.

`eks:CreateCluster` creates a public Kubernetes API endpoint in _another_ AWS account, one that you do not own.

Although by default the API requires an authorized token to perform sensitive actions[^777], it can still be hit by the Internet and does not need an IGW.

[^777]: By default only the `system:public-info-viewer` cluster role provides access to a set of endpoints for the `system:unauthenticated` group. These endpoints (e.g. `/healthz`, `/livez`, `/readyz`, and `/version`) are [used by Network Load Balancers to perform health checks](https://aws.amazon.com/blogs/security/how-to-use-new-amazon-guardduty-eks-protection-findings/).

I do not know of any other API calls like this.

It is likely I missed some, let me know via email and I'll update this.

For global accelerator, you still need a [symbolic IGW in the VPC](https://aws.amazon.com/blogs/networking-and-content-delivery/accessing-private-application-load-balancers-and-instances-through-aws-global-accelerator/), so I did not include it here.

To be clear, I am not talking about messing up an IAM trust policy or a [resource-based policy](https://matthewdf10.medium.com/aws-accounts-as-security-boundaries-97-ways-data-can-be-shared-across-accounts-b933ce9c837e) -- only what could show up from doing traditional network scanning / searching Shodan. 

For most AWS accounts, you should have an allowlist strategy for IAM service prefixes, that limit the amount of AWS services you need to know in-depth, and help know which obscure services your engineers are using.

For sandbox accounts, this is not feasible, so you will need to manually maintain a deny list of these IAM actions or services.[^1424]


[^1424]: Granted, you should be seamlessly re-creating sandbox accounts [or less ideally, nuking] regularly and only have public data in them.

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
- Did I get anything wrong? I'm sure the Internet will let me know in spades.

## Conclusion

Let me know how it goes limiting your Internet-exposed attack surface in an easy to understand, secure-by-default way. 

You might still get breached, but hopefully in a more interesting way.


![alt text](https://media.tenor.com/YrcU9HOzqmUAAAAC/majin-buu-clap.gif)

## Footnotes

   

