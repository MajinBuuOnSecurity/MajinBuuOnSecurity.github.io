---
layout: post
toc: true
title: "Preventing Accidental Internet-Exposure of AWS Resources"
---

Many AWS customers have suffered breaches due to exposuring resources to the Internet by accident. This post walks through all the different ways to mitigate that risk.


## About The Problem

In your multi-account AWS strategy, most accounts should have account-wide public block access for S3 enabled at account creation time and turned immutable via SCP. This prevents a whole class of issues. When a public S3 bucket is needed, separate accounts (under a separate OU than other accounts) should be made to host them.

How do you implement the same strategy for resources in a VPC (EC2 instances, ELBs, RDS databases, etc.)?


In AWS, Egress to the Internet is tightly coupled with Ingress from the Internet. This is because an Internet Gateway is necessary for both. In most cases, only the former is needed, to e.g. reach out to 3rd parties or update Linux packages.




## Options Towards Preventing by Design

It is important to note, that VPC Peering is not an option. See the FAQ for more information.

This will also save you NATGW money, which is a win for your partner teams. (TODO: Word this about persuasion.)


### Transit Gateway (TGW)

This is the most common way of implementing this.


#### A Note On Cost

On one hand, AWS states:

>Deploying a NAT gateway in every spoke VPC can become cost prohibitive because you pay an hourly charge for every NAT gateway you deploy (refer to Amazon VPC pricing), so centralizing it could be a viable option. 

On the other hand, 

>In some edge cases when you send huge amounts of data through NAT gateway from a VPC, keeping the NAT local in the VPC to avoid the Transit Gateway data processing charge might be a more cost-effective option.

Sending huge amounts of data through a NAT Gateway should be avoided anyway.
S3, Splunk, and [Honeycomb](https://docs.honeycomb.io/integrations/aws/aws-privatelink/) and similar companies have VPC endpoints. 





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


### IPv6 for Egress

You can do everything you normally do with IPv4, but 1


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

- But what about banning e.g Elastic IP Actions?
That is one mole you can whack, but there are so many others. 

- But what about VPC peering?
- How do I access machines if not public IP & IP whitelist, or DirectConnect->PrivateLink?


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


