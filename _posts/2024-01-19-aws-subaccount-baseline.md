---
layout: post
toc: true
title: "Security Baseline for AWS Subaccounts"
---

I wanted to create some basic open-source code to create new AWS accounts and Terraform them. Every company seems to re-implement this privately, this can serve as  a starting point for them. 

<!-- (It is also a prerequisite to other things I want to write about.) -->

## Summary



### How It Works

The steps are as follows:

1. Making the account
2. Creating the SCP baseline
3. Creating the account configuration baseline
4. Modifying existing Terraform

A lot of ink has been spilled on the 2nd Part, SCPs, but not how they fit into the bigger picture.

This is the foundation that lays the ground work for everything else. And the north star all legacy accounts and acquisitions get improved to match.

It is anologous to, in the AppSec realm, [a cookie-cutter repository](https://www.cookiecutter.io/templates) enabling all the security bells-and-whistles to new microservices. 

My initial goal is that companies can fork the repository to get started on their own custom baseline. A reference implementation.

Because all companies will have different OU structure, (think SCP inheritance), it is unlikely different companies will be able to use this same code. 



<!-- Perhaps one day the modules can be modular enough that different companies to use the same OSS code. -->
<!-- Hopefully other companies can contribute to it too. -->
<!-- n the UI/UX for customers or detailed to the extent I would like. -->
<!-- and What Does Good Look Like for your entire AWS footprint. -->

<!-- We will not cover e.g. east-west traffic. -->


## Handling All Those Services

Since each AWS service has its own nuances and possible security problems. The primary way to mitigate this risk is to [implement an allowlist](#implementing-the-allowlist) strategy for IAM services, which limits the amount of AWS services you need to know in-depth.

Once you have this allowlist, you can cross-refence each service with the [Mitigations Per Service](#mitigations-per-service) section and ensure you have the proper mitigations in place.

### Implementing the Allowlist

When a customer requests an AWS account[^1424], the fill-out form should ask them a set of questions including, "Which AWS services and regions will you be using?"

[^1424]: For sandbox accounts, this may not be realistic, so you may need to manually maintain a deny list of dangerous services. Granted, you should be seamlessly re-creating sandbox accounts (or, less ideally, [nuking](https://github.com/rebuy-de/aws-nuke)) regularly and only have public data in them.

This will end up translating to e.g. Terraform local variables
```go
locals {
  services = ["dynamodb", "ec2", "s3"]
  regions = ["us-east-2"]
}
```

Which will further be passed into a configuration module, an SCP module, or both, for the subaccount.

### Service-Specific Mitigations

Rather than simply giving the subaccount admin IAM role `"ec2:*"` and calling it a day, you should go further and ask service-specific questions.

- "Do you need publicly accessible EC2 resources? If so, please explain why."
- "Can you use our Terraform module for launching EC2? If not, please explain why."

Which can, depending on the answers, end up getting translated to e.g.
```go
module "project_x_account" {
  source = "../../modules/subaccounts/configuration"

  services = local.services
  regions = local.regions

  imdsv1_disabled = false  // AWSSEC-123 TODO: Remove this
}
```
If for some reason the customer needed IMDSv1 enabled in their account.

The `imdsv1_disabled` argument, will turn-off a mitigation that is on by default, such as an [SCP enforcing IMDSv2](https://github.com/ScaleSec/terraform_aws_scp/blob/521ac29d712a6ebb51feb6f11b56e6c40b61bada/security_controls_scp/modules/ec2/require_imdsv2.tf#L5-L28) through the `"ec2:MetadataHttpTokens"` [condition key](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html#amazonec2-policy-keys).


## FAQ

### What does this not solve?

- AWS Account Inventory System with a good UI.

Tags are not a great solution for storing metadata.

- A Web UI for Creating the Account

I could build a minimal Flask app to ask customers to fill out a form, and then call the AWS API myself to make the account.

The web app would need to be integrated with GitHub to make the PR, however.



### What about Buy instead of Build here?

That is a rabbit hole of a subject. My bottom line up front (BLUF) is that this should satisfy your every need.

If it doesn't I'd be interested to know why. I'll (eventually, probably months later) update this post with your feedback.

Compared to [Account Factory](https://docs.aws.amazon.com/controltower/latest/userguide/aft-architecture.html), running a Python script is much more manual, a Flask app would improve the UI as I answer in the previous question. 

However, AF is complicated AF, see the [Rude Goldberg machine](https://en.wikipedia.org/wiki/Rube_Goldberg_machine) diagram:
![alt text](https://i.imgur.com/J13xltK.png)


### How do I translate this work into metrics on my resume?

For this part, "baseline" is an ever moving target. There is no industry standard baseline. Since this is OSS you can reference it, however.

Story time:
Ryan McHeegan wrote a blameless post-mortem of the Mudge disclosure [^2]
Present to the Twitter board that 90% of Mac laptops now adhere to the baseline.
Where all it is is there is a program installed on 90% of laptops. 

[^2]: Two people I have enormous amounts of respect for, goes without saying.
