---
layout: post
toc: true
title: "Preventing Accidental Internet-Exposure of AWS Resources (Part 3: Tech Debt Mountain)"
---

When addressing a vast, monolithic account housing a blend of private and public assets, how can you rectify the absence of preventative design principles?

![alt text](https://i.imgur.com/t0Kx9jS.png)

## IAM Protection Rings

Think about your IAM roles as [privilege rings](https://en.wikipedia.org/wiki/Protection_ring) in x86:

<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/2/2f/Priv_rings.svg/1350px-Priv_rings.svg.png" alt="drawing" width="500"/>

In AWS, rings 1 through 3 can be different types of IAM roles:

- Ring 0 is SCPs
- Ring 1 can make public-facing resources
- Ring 2 needs to e.g. create load balancers, but none should be Internet-facing.
- Ring 3 needs to e.g. create EC2 instances, but none should be Internet-facing.

With an SCP banning `ec2:CreateInternetGateway`, rings 1 through 3 cannot create Internet-facing assets no matter what, as ring 0 prevents them -- but without this, we need to play IAM games resembling [whack-a-mole](https://www.youtube.com/watch?v=iqihTaHblzM).

![alt text](https://media.tenor.com/hGclJ34JeSIAAAAC/one-punch.gif)

Next we will dive deep into one use-case, then list all use-cases, then summarize how to solve them.

### Supporting Ring 3 with RunInstances

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

## How It Happens

Let's see if we can list every possible mole that can pop up.

Creation of load balancer in public subnet:

![alt text](https://i.imgur.com/GarWehc.gif)

Creation of instance in public subnet:

![alt text](https://i.imgur.com/1e4M8z4.gif)

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

This is not much of an issue, but if can prevent it 'for free' anyway: Creation of public instance in private subnet using `ec2:AssociatePublicIpAddress`

![alt text](https://i.imgur.com/iWE1bzH.gif)

## How To Stop It

Banning IAM Actions
- Cannot create Load Balancers
- Cannot create EIPs
- Cannot modify target groups
- Cannot associate EIPs

Banning IAM Actions Conditionally
- Cannot touch public subnets
- Cannot `ec2:RunInstances` with 
- Cannot modify target groups of public load balancers

## Footnotes

   

