---
layout: post
toc: true
title: "[Rough Draft] Preventing Accidental Internet-Exposure of AWS Resources (Part 2: Handling All Services)"
---

While having Private VPC accounts is a great first step -- it is only a partial solution -- since there are many AWS services that do not require a VPC to make a resource public through network access.

There are hundreds of AWS services -- nobody knows them all -- so what do you do?

## Handling All Those Services

Since each AWS service has its own nuances and possible security problems. The primary way to mitigate this risk is to [implement an allowlist](#implementing-the-allowlist) strategy for services, which limits the amount of services you need to know in-depth.

Once you have this allowlist, you can cross-reference each service with the [Mitigations Per Service](#mitigations-per-service) matrix below, and ensure you have the proper mitigations in place.

## Implementing the Allowlist

When a customer requests an AWS account[^1424], the fill-out form should ask them a set of questions including, "Which AWS services and regions will you be using?"

[^1424]: For sandbox accounts, this may not be realistic, so you may need to manually maintain a deny list of dangerous services. Granted, you should be seamlessly re-creating sandbox accounts (or, less ideally, [nuking](https://github.com/rebuy-de/aws-nuke)) regularly and only have public data in them.

This can get fed into whatever your account creation automation you have. As a concrete example, let's say we end up translating to e.g. [Terraform local variables](https://github.com/MajinBuuOnSecurity/Terraform-Monorepo/blob/main/subaccounts/smoky-production/configuration_baseline/locals.tf):
```go
locals {
  services = ["dynamodb", "ec2", "lambda," "s3"]
  regions = ["us-east-2"]
}
```

Which will further be passed into a [configuration module](https://github.com/MajinBuuOnSecurity/Terraform-Monorepo/tree/main/modules/subaccount_baselines/configuration), an [SCP module](https://github.com/MajinBuuOnSecurity/Terraform-Monorepo/tree/main/modules/subaccount_baselines/scps), or both, for the subaccount -- in order to establish a security baseline.

### Service-Specific Mitigations

Rather than simply giving the subaccount [admin IAM role](https://github.com/MajinBuuOnSecurity/Terraform-Monorepo/blob/df5a1021fb1ff6975d394ade7081050472f0e8ff/automation/create_aws_account/organizations.py#L23) `"lambda:*"` and calling it a day, you should go further and ask service-specific questions:
- "Will all Lambda functions be deployed in one of our VPCs?"
- "Will all Lambda functions require [AWS_IAM authentication](https://docs.aws.amazon.com/lambda/latest/dg/urls-auth.html#urls-auth-iam)?"
- "Can you use our [company-specific] Terraform module for creating these Lambda functions?"

Which can, if for some reason the customer needs to have publicly accessible Lambda function URLs, end up getting translated to:
```go
module "project_x_account" {
  source = "../../modules/subaccounts/scps"

  services = local.services
  regions = local.regions

  deny_auth_type_not_iam_lambda = false
}
```


The `deny_auth_type_not_iam_lambda` argument, will turn-off a mitigation that is on by default, [via the module](https://github.com/MajinBuuOnSecurity/Terraform-Monorepo/blob/df5a1021fb1ff6975d394ade7081050472f0e8ff/modules/subaccount_baselines/scps/AccountSCP.tf#L23-L65):
```go
    {
      include = var.deny_auth_type_not_iam_lambda,
      effect = "Deny"
      actions = [
        "lambda:CreateFunctionUrlConfig",
        "lambda:UpdateFunctionUrlConfig",
      ]
      resources = ["arn:aws:lambda:*:*:function/*"]
      conditions = [
        {
          test     = "StringNotEquals"
          variable = "lambda:FunctionUrlAuthType"

          values = [
            "AWS_IAM",
          ]
        },
      ]
    },
```

## Mitigation Types

For preventing public network access, there are 4 possible account-wide mitigation types per service:

- Don't have public subnets
- Don't have an Internet Gateway (IGW)
- Use a condition key
- Use a condition key in combination with resource type limits

It is important to note <ins>we are at the mercy of AWS as to which mitigations are available/applicable for each specific service.</ins>

Ideally, every service with resources created outside of a VPC would have a condition key (similar to `lambda:FunctionUrlAuthType` above) we could block and then we would have solid [security invariants](https://www.chrisfarris.com/post/philosphy-of-prevention/). Unfortunately, this is not the case for most services.

#### No Public Subnets

As long as there are no public subnets in an account, the service cannot have Internet-facing resources.

[ELB v1 / v2](#elb-v1--v2) is an example.

#### No IGWs

As long as there are no IGWs in an account, the service cannot have Internet-facing resources.

[Global Accelerator](#global-accelerator) is an example.

#### Condition Key

There is a specific IAM condition key, that under some condition, can be blocked.

[Lambda](#lambda) is an example.

#### Condition Key + Resource Type Limits

Both a condition key needs to be used, as well as limiting the types of resources to which that condition key is applied.

[API Gateway](#api-gateway) is an example.

## No-Mitigations-Available Services

These are services where the 4 above mitigation types are completely useless for preventing public network access.

[EKS](#eks) is an example.

### The Solution for Services with No Mitigations

If your customers need `eks:CreateCluster`, then you need to rely on a trusted IAM role.

(As well as alerting, but that is reactive.)



## Mitigations Per Service

Service                                             | No Public Subnets | No IGW  | Condition Key | Condition Key + Resource Type Limits | Need To Ban
----------------------------------------------------|-------------------|---------|---------------|--------------------------------------|------------
<span style="color:IndianRed">API Gateway</span>    | N/A               | N/A     | Partial       | Yes                                  | No
<span style="color:green">Athena</span>             | N/A               | N/A     | N/A           | N/A                                  | No
CloudFront                                          | ?                 | ?       | ?             | ?                                    | ?
<span style="color:green">DynamoDB</span>           | N/A               | N/A     | N/A           | N/A                                  | No
ECS                                                 | ?                 | ?       | ?             | ?                                    | ?
<span style="color:IndianRed">EC2</span>            | Yes-ish           | Yes     | Partial       | N/A                                  | No
<span style="color:Maroon">**EKS**</span>               | Partial           | Partial | No            | N/A                                  | <span style="color:Maroon">**Yes**</span> 
<span style="color:green">ElasticCache</span>       | N/A               | N/A     | N/A       | N/A                                  | No
<span style="color:IndianRed">ELB v1 / v2</span>    | Yes               | Yes     | No            | N/A                                  | No
EMR                                                 | ?                 | ?       | ?             | ?                                    | ?
<span style="color:IndianRed">Global Accelerator</span>| No                | Yes     | No            | No                                   | No
<span style="color:IndianRed">Lambda</span>            | N/A               | N/A     | Yes           | N/A                                  | No
Lightsail                                           | ?                 | ?       | ?             | ?                                    | ?
Neptune                                             | ?                 | ?       | ?             | ?                                    | ?
<span style="color:IndianRed">RDS</span>               | Yes-ish           | Yes     | No            | N/A                                  | No
<span style="color:IndianRed">Redshift</span>          | Yes-ish           | Yes     | No            | N/A                                  | No
<span style="color:green">S3</span>                 | N/A               | N/A     | N/A           | N/A                                  | No
<span style="color:green">SNS</span>                | N/A               | N/A     | N/A           | N/A                                  | No
<span style="color:green">SQS</span>                | N/A               | N/A     | N/A           | N/A                                  | No

TBD:
- CloudFront
- ECS
- EMR
- Lightsail
- Neptune

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

Note: This takes the cake for weirdest and most error prone IAM example I have ever seen. If you can turn this into a Deny statement somehow I'll give you a prize.

### CloudFront

???

### ECS

???

### ELB v1 / v2

Load-balancers with scheme "internet-facing" can only exist in public subnets, this is enforced at creation time.

TODO: What would happen if I changed the route table?

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

Summary: No IGW, no problem.

Same as EC2.

### Redshift

Summary: No IGW, no problem.

The ElasticIp argument in both [CreateCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_CreateCluster.html) and [ModifyCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_ModifyCluster.html) says,
"The cluster must be provisioned in EC2-VPC [as oppsosed to EC2-Classic, I assume] and publicly-accessible through an Internet gateway."

_Based on this, I am concluding that no IGW / no public subnets] would make it so that you cannot access a Redshift cluster. I am not testing this, however, since it would be too expensive._ EC2 does not require an instance be in a public subnet to assign an EIP to it, so it would be odd for Redshift to, however RDS [documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.DBInstance.Modifying.html) says something [similar](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html#USER_VPC.Hiding).

As further evidence, [Instructions for turning a private cluster public](https://repost.aws/knowledge-center/redshift-cluster-private-public) state: "Note: An Elastic IP address is required. If you do not choose one, an address will be randomly assigned to you."

In only the [ModifyCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_ModifyCluster.html) documentation it states: "Only clusters in VPCs can be set to be publicly available." The [CreateCluster](https://docs.aws.amazon.com/redshift/latest/APIReference/API_CreateCluster.html) documentation does not state this.

[No condition keys](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonredshift.html#amazonredshift-policy-keys) exist for e.g. `Encrypted` or `ElasticIP` or `PubliclyAccessible`.

## Footnotes
