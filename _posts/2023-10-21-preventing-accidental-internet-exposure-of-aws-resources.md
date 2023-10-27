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

When a public S3 bucket is needed, ideally it would be made in a special `S3 Public Resources` account (under a separate OU) where only public S3 buckets can live.[^2]

[^2]: [New S3 buckets have Public Access Block by default now](https://aws.amazon.com/about-aws/whats-new/2022/12/amazon-s3-automatically-enable-block-public-access-disable-access-control-lists-buckets-april-2023/), but being able to look at your AWS Org structure from a thousand-foot view and know which subtree can have public S3 buckets is invaluable.

A simplified and idealistic[^2154] AWS organization structure supporting this is:

![alt text](https://i.imgur.com/bPIKZoC.png)

[^2154]: You may think this architecture is unrealistic for your company, and you might be right.

## About The Problem

The question this post answers is: How do you implement the same strategy for resources in a VPC (EC2 instances, ELBs, RDS databases, etc.)?

These are resources that can be found by an attacker via traditional public IP network scanning or [searching Shodan](https://maia.crimew.gay/posts/how-to-hack-an-airline/).


This is more complicated, because in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

The reason they are tightly coupled: is an Internet Gateway (IGW) is necessary for both. So if an engineer has IAM permissions to create an IGW, or is unrestrained in how they create resources in a VPC that has one, they can expose resources to the Internet.

The Egress use-case typical looks like:
![alt text](https://i.imgur.com/vKsdNOh.png)

Whereas an accidental Internet exposure might look like:

![alt text](https://i.imgur.com/1e4M8z4.gif)

Or:

![alt text](https://i.imgur.com/gyXZz2E.gif)

There are many ways to expose resources to the Internet like this, but the key insight is that they all require an Internet Gateway (IGW).

## Preventing by Design

Since an IGW is the root of all of our problems, we simply can ban their creation via SCP 
(`"ec2:CreateInternetGateway"`) upon vending an account, (alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.)

That's it. Problem solved! You can hand the subaccount over to the customer, they will never be able to make public-facing load balancers or EC2 instances regardless of their IAM permissions.

You can then, put accounts that were vended this way with the IGW-deny-SCP, in an OU and achieve [What Good Looks Like](#what-good-looks-like) above.

But what if you need to support the Egress use-case above?

Then you need to ensure your network architecture tightly couples a NAT Gateway with an Internet Gateway, by e.g. giving subaccounts a paved path to a NAT Gateway in another account, possible ways to do this are:

1. [Centralized Egress via Transit Gateway (TGW)](#option-1-centralized-egress-via-transit-gateway-tgw)
2. [Centralized Egress via PrivateLink (or VPC Peering) with Egress Filtering](#option-2-centralized-egress-via-privatelink-or-vpc-peering-with-egress-filtering)
3. [VPC Sharing](#option-3-vpc-sharing)
4. [IPv6 for Egress](#option-4-ipv6-for-egress)

Hopefully one of these options will align with the goals of your networking team. 

My personal recommendation is: Go with TGW. Once you are ready to implement Egress Filtering, have assets use PrivateLink + Egress Filtering.

### Option 1: Centralized Egress via Transit Gateway (TGW)

This is the most common implementation, and probably the best. If money is no issue for you: go this route.

It looks like this:

![alt text](https://i.imgur.com/alRH2hN.png) 

AWS first wrote about this [in 2019](https://aws.amazon.com/blogs/networking-and-content-delivery/creating-a-single-internet-exit-point-from-multiple-vpcs-using-aws-transit-gateway/)[^5555] and lists it under their prescriptive guidance as [Centralized Egress](https://docs.aws.amazon.com/prescriptive-guidance/latest/transitioning-to-multiple-aws-accounts/centralized-egress.html).

As you can see, each VPC in a subaccount has a route table that has 0.0.0.0/0 destined traffic sent to a TGW in another account, where an IGW does live.

[^5555]: Let me know if you find an earlier reference.

#### A Note On Cost

This will reduce your NAT Gateway cost, which is arguably the most notoriously expensive networking component in AWS.

On one hand, [AWS states](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/centralized-egress-to-internet.html):

>Deploying a NAT gateway in every AZ of every spoke VPC can become cost-prohibitive because you pay an hourly charge for every NAT gateway you deploy, so centralizing it could be a viable option.

On the other hand, they also say:

>In some edge cases, when you send huge amounts of data through a NAT gateway from a VPC, keeping the NAT local in the VPC to avoid the Transit Gateway data processing charge might be a more cost-effective option.

Sending huge amounts of data through a NAT Gateway should be avoided anyway.
[S3](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html), [Splunk](https://www.splunk.com/en_us/blog/platform/announcing-aws-privatelink-support-on-splunk-cloud-platform.html), [Honeycomb](https://docs.honeycomb.io/integrations/aws/aws-privatelink/), and similar companies[^1350] have VPC endpoints you can utilize to lower NAT Gateway data processing charges.

[^1350]: There are some companies with agents meant to be deployed on EC2s that do not offer a VPC endpoint, [perhaps](https://sso.tax/) a [wall](https://fido.fail/) of [shame](https://github.com/SummitRoute/imdsv2_wall_of_shame#imdsv2-wall-of-shame) can be made.

### Option 2: Centralized Egress via PrivateLink (or VPC Peering) with Egress Filtering

PrivateLink and VPC Peering are mostly non-options. See the [FAQ](#why-is-vpc-peering-not-a-straightforward-option) for more information.

However, if you are willing to do a lot of heavy lifting that is orthogonal to AWS primitives, you _can_ use these in combination with [Internet Egress Filtering](https://eng.lyft.com/internet-egress-filtering-of-services-at-lyft-72e99e29a4d9), to accomplish centralized egress. This is because the destination IP of outbound traffic won't be the Internet, but a private IP.

This is assuming you are deploying e.g. iptables, to re-route Internet-destined traffic on every host.

Some reasons you may not want to do this are:
- Significant effort
- It won't be possible for all subaccount types, such as sandbox accounts. (Where requiring `iptables` and a proxy are too heavyweight.)
- Egress filtering is lower-priority than preventing accidental Internet-exposure. So tightly coupling the two and needing to setup a proxy first may not make strategic sense.
- If something goes wrong on the host, the lost traffic will not appear in VPC flow logs [^98] or traffic mirroring logs.[^985] The DNS lookups will show up in Route53 query logs, but that's it.

With that said, AWS does not have a primitive to perform Egress filtering [^99], so you will eventually have to implement Egress filtering via a proxy. Therefore, in production accounts, you can go with this option. For sandbox accounts, use a different centralized egress pattern e.g. VPC sharing (which will not disrupt an org migration due to their ephemeral nature).

Using PrivateLink, in this way, would look like this:

![alt text](https://i.imgur.com/5Vh2SuX.png)

[^98]: [Flow log limitations](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html#flow-logs-limitations) does not state "Internet-bound traffic sent to a peering connection" or "Internet-bound traffic sent to a VPC interface endpoint." under `The following types of traffic are not logged:`. After testing I believe it should, but these are likely omitted due to not being a proper use-case.

[^985]: A peering connection cannot be selected as a [traffic mirror source or target](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-targets.html), but a network interface can. However, only an ENI belonging to an EC2 instance can be a mirror source, not an ENI belonging to an Interface endpoint. The documentation doesn't mention this anywhere I could find.

[^99]: It has [AWS Network Firewall](https://aws.amazon.com/network-firewall/faqs/), which can be fooled via SNI spoofing. So it is, at best, a stepping stone to keep an inventory of your Egress traffic if you can’t get a proxy up and running short-term and are not using TLS 1.3 with encrypted client hello (ECH) or encrypted SNI (ESNI).


### Option 3: VPC Sharing

This is a tempting, simple option, that is not well known.[^996]

You can simply make a VPC in your networking account, and share private subnets to subaccounts.

The problem with this approach, is that there will _still be an Internet Gateway in the VPC_, if you do not go with one of the other approaches.

Being limited to private subnets helps with 2 problems:

1. Engineers can launch EC2 instances and you won't have to worry about them implicitly getting a public IP address
2. Engineers can create Internet-facing Load Balancers, as `CreateLoadBalancer` requires that a public subnet be provided

An engineer can still make Internet-facing assets with e.g.

- The `ec2:AssociatePublicIpAddress` condition key
- Global Accelerator
- Any future services that treat the presence of an IGW as a welcome mat


TODO: Talk about the extra steps here.. TODO

Unfortunately, you cannot leverage a TODO

You cannot create an Internet-facing asset in a private subnet, you can however setup `ec2:AssociatePublicIpAddress` / Global Accelerator.

![alt text](https://i.imgur.com/4OojfzP.png)

[^996]: Shout out to [Aiden Steele](https://awsteele.com/blog/2022/01/02/shared-vpcs-are-underrated.html) and [Stephen Jones](https://sjramblings.io/unlock-the-hidden-power-of-vpc-sharing-in-aws) for writing their thoughts on VPC Sharing.

#### Why a NACL Cannot Help

A NACL is a stateless TODO

The traditional TODO


#### A Strategic Implication of VPC Sharing

In the [Shareable AWS Resources](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html#shareable-vpc) page of the AWS RAM documentation, `ec2:Subnet` is one of 7 resource types marked as

> Can share with ***only*** AWS accounts in its own organization.

This means you can never perform an AWS organization migration in the future.

If you are like most AWS customers:
- The Management Account of your AWS Organization has most of your resources in it
- You want to follow Best Practices[^996221] and have an empty Management Account
- It is infeasible to 'empty' out the current management account over time

[^996221]: This is Stage 1 of Scott's [AWS Security Maturity Roadmap](https://summitroute.com/downloads/aws_security_maturity_roadmap-Summit_Route.pdf), for example.

Then you will need to perform an org migration in the future, and should stay away from VPC Sharing.

However, suppose you are willing to risk a production outage 😂 (Particularly if you do not use ASGs or Load Balancers, which will lose access to the subnets.) The NAT Gateway will remain functional according to AWS.

See

>Scenario 5: VPC Sharing across multiple accounts

from [Migrating accounts between AWS Organizations, a network perspective](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-accounts-between-aws-organizations-from-a-network-perspective/) by Tedy Tirtawidjaja for more information.

### Option 4: IPv6 for Egress

Remember how I said, "in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet" above. For IPv6, that's a lie.

IPv6 addresses are globally unique and, therefore, public by default. Due to this, AWS created the primitive of an [Egress-only Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html), 

Unfortunately, with this primitive, there is no way to connect IPv4-only destinations, so if that is necessary -- which it likely is -- go with one of the other 3 options.

![alt text](https://i.imgur.com/wtuaa71.png)

#### More details around IPv4-only Destinations

As Sébastien Stormacq wrote in [Let Your IPv6-only Workloads Connect to IPv4 Services](https://aws.amazon.com/blogs/aws/let-your-ipv6-only-workloads-connect-to-ipv4-services/), you need only add a route table entry and set `--enable-dns64` on subnets to accomplish this -- but you unfortunately still need an IGW.

DNS queries made to the Amazon-provided DNS Resolver in subnets will then return synthetic IPv6 addresses for IPv4-only destinations with the well-known `64:ff9b::/96` prefix. And due to the route table entry, traffic with that prefix will be sent to the NAT Gateway.

The problem is the NAT Gateway then needs an IGW to communicate with the destination, so one of the [other options](https://d1.awsstatic.com/architecture-diagrams/ArchitectureDiagrams/IPv6-reference-architectures-for-AWS-and-hybrid-networks-ra.pdf) becomes necessary.

### Tradeoffs

Criteria                   | TGW                   | VPC Sharing           | IPv6-Only             | PrivateLink + Egress Filtering
-------------------------- | --------------------- | --------------------- | --------------------- | ---------------------
AWS Billing Cost           | <span style="color:red">Highest</span> | Lowest                              | Low                                   | Medium
Complexity*                | Medium                                 | Low                                 | Medium                                | <span style="color:red">High</span>
Scalability*               | High                                   | Low                                 | Medium                                | High
Flexibility*               | High                                   | Medium                              | <span style="color:red">Lowest</span> | High
Will Prevent Org Migration | False                                  | <span style="color:red">True</span> | False                                 | False

\* = YMMV

## Stopping The Bleeding

What if you have a giant monolithic account with a mix of private and public assets to which you didn’t apply these design principles?

All hope is not lost, but you and I will need to play [whack-a-mole](https://www.youtube.com/watch?v=iqihTaHblzM) together.

![alt text](https://media.tenor.com/hGclJ34JeSIAAAAC/one-punch.gif)

### IAM Protection Rings

Think about your IAM roles as [privilege rings](https://en.wikipedia.org/wiki/Protection_ring) in x86:

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/1350px-Priv_rings.svg.png" alt="drawing" width="500"/>

In AWS, rings 1 through 3 can be different types of IAM roles:

- Ring 0 is SCPs
- Ring 1 can make public-facing resources
- Ring 2 needs to e.g. create load balancers, but none should be Internet-facing.
- Ring 3 needs to e.g. create EC2 instances, but none should be Internet-facing.

With an SCP banning `ec2:CreateInternetGateway`, rings 1 through 3 cannot create Internet-facing assets no matter what, as ring 0 prevents them -- but without this, we need to play IAM games.

#### Supporting Ring 3 with RunInstances

For Ring 3, we should be able to allow the role to call `ec2:RunInstances`, under the condition that [`ec2:AssociatePublicIpAddress`](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html#amazonec2-policy-keys) is `false`. 

Except that is insufficient: if the [subnet given](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/run-instances.html) has [`"map-public-ip-on-launch"`](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/modify-subnet-attribute.html#options) set to true, the EC2 will get a public IP.

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

So, we need to write [our own custom Terraform provider](https://gist.github.com/MajinBuuOnSecurity/cb6b4689b47f555a2324c3f33da8e7eb#file-public_subnets-go-L172-L181) to iterate through all regions.

[^1404]: Even if it did, [`"aws_subnets"`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/subnets) is one of the [better](https://github.com/hashicorp/terraform/issues/16380#issuecomment-418476841) data sources. E.g. [aws_lb](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/lb) and [aws_lbs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/lbs) do not have filters, and the latter fails if there are none.

Finally, we can create an [`aws_iam_policy_document`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document)[^1427] similar to the following abbreviated[^1428] policy:

[^1427]: See [this gist](https://gist.github.com/MajinBuuOnSecurity/205273d308435f8d4a52759836aeb9e5) for a denylist example.
[^1428]: If going with an allowlist approach, you also need statements that cover all the other resource types (image, instance, security-group, volume etc.)


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

Now, when someone in Ring 1 or 2 creates a new subnet, they can also apply this dynamic policy so Ring 3 can automatically launch instances in it.

This is  <font size="+2"><strong>one mole</strong></font> we just whacked. There are many others. This is What Bad Looks Like.

### How It Happens

Let's see if we can list every possible mole that can pop up.

Creation of load balancer in public subnet:

![alt text](https://i.imgur.com/GarWehc.gif)

Creation of instance in public subnet:

![alt text](https://i.imgur.com/1e4M8z4.gif)

Creation of public instance in private subnet using `ec2:AssociatePublicIpAddress`:

![alt text](https://i.imgur.com/iWE1bzH.gif)

EC2 added as Target group of load balancer

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

## Additional Mitigations Required

Sadly there are many AWS services that do not require an IGW to make a resource public through network access.

The primary way to mitigate this is to deploy an allowlist strategy for IAM service prefixes, this limits the amount of AWS services you need to know in-depth.

For sandbox accounts, this may not be realistic, so you will need to manually maintain a deny list of these IAM actions.[^1424]

[^1424]: Granted, you should be seamlessly re-creating sandbox accounts (or, less ideally, nuking) regularly and only have public data in them.

See [https://github.com/SummitRoute/aws_exposable_resources#resources-that-can-be-made-public-through-network-access](https://github.com/SummitRoute/aws_exposable_resources#resources-that-can-be-made-public-through-network-access) for a high-level list of actions. I'd encourage you to cut a PR to that repository if you know of a service that is not listed there, I will put extra details in this post about those actions.

### Mitigations Per Service

Service          | No Public Subnets | No IGW  | Condition Key | Condition Key + Resource Type Limits | Need To Ban
-----------------|-------------------|---------|---------------|--------------------------------------|------------
API Gateway      | N/A               | N/A     | Partial       | Yes                                  | No
CloudFront       | ?                 | ?       | ?             | ?                                    | ?
ECS              | ?                 | ?       | ?             | ?                                    | ?
EC2              | Partial           | Yes     | Partial       | N/A                                  | No
EKS              | Partial           | Partial | No            | N/A                                  | Yes
ELB v1 / v2      | Yes               | Yes     | No            | N/A                                  | No
EMR              | ?                 | ?       | ?             | ?                                    | ?
Lambda           | N/A               | N/A     | Yes           | N/A                                  | No
Lightsail        | ?                 | ?       | ?             | ?                                    | ?
Neptune          | ?                 | ?       | ?             | ?                                    | ?
RDS              | ?                 | ?       | ?             | ?                                    | ?
Redshift         | Maybe             | Yes     | No            | N/A                                  | No

Additionally, as [Arkadiy Tetelman](https://github.com/arkadiyt) previously [noted](https://github.com/arkadiyt/aws_public_ips/blob/bb973055c1b8a14af6dbb057aa26cfe0a2ab47c9/lib/aws_public_ips/cli.rb#L5-L39),the following services are managed by AWS such that they are of no concern when it comes to network accessible endpoints:
- Athena
- DynamoDB
- ElasticCache
- S3
- SNS
- SQS

### API Gateway

Summary: Either ban the whole service or have a limited allow solely for private REST APIs.

Only REST APIs can be private, [HTTP](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html) and [Websocket](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api.html) APIs cannot be.

So you have to [limit the allowed resources to REST APIs, and ensure `"apigateway:Request/EndpointType"` is private](https://docs.aws.amazon.com/apigateway/latest/developerguide/security_iam_id-based-policy-examples.html#security_iam_id-based-policy-examples-private-api):
```
"Resource": [
"arn:aws:apigateway:us-east-1::/restapis",
"arn:aws:apigateway:us-east-1::/restapis/??????????"
],
...
"ForAllValues:StringEqualsIfExists": {
  "apigateway:Request/EndpointType": "PRIVATE",
  "apigateway:Resource/EndpointType": "PRIVATE"
}
```

Note: This takes the cake for weirdest and most error prone IAM example I have ever seen. If you can turn this into a Deny statement some how I'll give you a prize.

### CloudFront

???

### ECS

???

### EKS

Summary: Ban the `eks:CreateCluster` action

`eks:CreateCluster` creates a public Kubernetes API endpoint in _another_ AWS account you do not own.

Although, by default, the API requires an authorized token to perform sensitive actions[^777], it can still be hit by the Internet and does not need an IGW.

[^777]: By default only the `system:public-info-viewer` cluster role provides access to a set of endpoints for the `system:unauthenticated` group. These endpoints (e.g. `/healthz`, `/livez`, `/readyz`, and `/version`) are [used by Network Load Balancers to perform health checks](https://aws.amazon.com/blogs/security/how-to-use-new-amazon-guardduty-eks-protection-findings/).


#### Sidenote About the EKS Cluster Role

Before creating a cluster, you must have a cluster IAM role with the  `AmazonEKSClusterPolicy` AWS managed policy attached.

This is an [overpermissive one-size-fits-all policy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonEKSClusterPolicy.html#AmazonEKSClusterPolicy-json).

You _can_ use a combination of `NotAction` and `Deny` to limit your EKS cluster role to what is actually needed.

One of the more dangerous permissions is `CreateLoadBalancer`.

If you already are limited to private subnets or have no IGWs, then it cannot create public-facing load balancers. As an alternative however, in a pinch you can bring your own load balancer[^8302] to EKS, rather than have the [`AWS Load Balancer Controller`](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html) create them.

[^8302]: TODO: Find the link about this.

This will enable you to limit operators of the EKS cluster to [Ring 3 style access](#iam-protection-rings).

### ElasticCache

All ElasticCache instances are private and designed to be used internally to your VPC, so without e.g. using an EC2 as a NAT instance, there is no concern.

### EMR

???

### Global Accelerator

For global accelerator, you still need a [symbolic IGW in the VPC](https://aws.amazon.com/blogs/networking-and-content-delivery/accessing-private-application-load-balancers-and-instances-through-aws-global-accelerator/).


### Lambda

Summary: There is a `FunctionUrlAuthType` [condition key](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awslambda.html#awslambda-policy-keys).

Block
```
"StringEquals": {
  "lambda:FunctionUrlAuthType": "NONE"
}
```
or more specifically
```
"StringNotEquals": {
    "lambda:FunctionUrlAuthType": "AWS_IAM"
}
```
to prevent the function URL endpoint will be public unless you implement your own authorization logic in your function."


Note: [Requiring a lambda lives in a customer-owned VPC](https://github.com/ScaleSec/terraform_aws_scp/blob/521ac29d712a6ebb51feb6f11b56e6c40b61bada/security_controls_scp/modules/lambda/require_vpc_lambda.tf#L9-L33) only affects Egress, not Ingress. So it is irrelevant here.

### Lightsail

???

### Neptune

???

### RDS

???

### Redshift

Summary: No IGW, no problem.

The ElasticIp argument in both [CreateCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_CreateCluster.html) and [ModifyCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_ModifyCluster.html) says,
"The cluster must be provisioned in EC2-VPC [as oppsosed to EC2-Classic, I assume] and publicly-accessible through an Internet gateway."

_Based on this, I am concluding that no IGW [or possibly no public subnets] would make it so that you cannot access a Redshift cluster. I am not testing this, however, since it would be too expensive._ EC2 does not require an instance be in a public subnet to assign an EIP to it, so it would be odd for Redshift to, however RDS documentation says something [similar](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html#USER_VPC.Hiding).

As further evidence, [Instructions for turning a private cluster public](https://repost.aws/knowledge-center/redshift-cluster-private-public) state: "Note: An Elastic IP address is required. If you do not choose one, an address will be randomly assigned to you."

In only the [ModifyCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_ModifyCluster.html) documentation it states: "Only clusters in VPCs can be set to be publicly available." The [CreateCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_CreateCluster.html) documentation does not state this.

[No condition keys](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonredshift.html#amazonredshift-policy-keys) exist for e.g. `Encrypted` or `ElasticIP` or `PubliclyAccessible`.

## FAQ

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

If you try to disable "[Source/destination checking](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#eni-basics)" (which ensures that the ENI is either the source or the destination of any traffic it receives), you will not be able to. So traffic is dropped before it would ever travel cross-account.

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

   

