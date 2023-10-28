---
layout: post
toc: true
title: "Preventing Accidental Internet-Exposure of AWS Resources (Part 2: Beyond EC2)"
---

There are many AWS services that do not require a VPC to make a resource public through network access. Let's walk through a bunch of different services and list the corresponding mitigations you need to put in place for each of them.

## Additional Mitigations Required

Sadly there are many AWS services that do not require an IGW to make a resource public through network access.

The primary way to mitigate this is to deploy an allowlist strategy for IAM service prefixes, this limits the amount of AWS services you need to know in-depth.

For sandbox accounts, this may not be realistic, so you will need to manually maintain a deny list of these IAM actions.[^1424]

[^1424]: Granted, you should be seamlessly re-creating sandbox accounts (or, less ideally, nuking) regularly and only have public data in them.

See [https://github.com/SummitRoute/aws_exposable_resources#resources-that-can-be-made-public-through-network-access](https://github.com/SummitRoute/aws_exposable_resources#resources-that-can-be-made-public-through-network-access)[^11245] for a high-level list of actions. I'd encourage you to cut a PR to that repository if you know of a service that is not listed there, I will put extra details in this post about those actions.

[^11245]: Special thanks to [Arkadiy Tetelman](https://github.com/arkadiyt) too, who previously [noted](https://github.com/arkadiyt/aws_public_ips/blob/bb973055c1b8a14af6dbb057aa26cfe0a2ab47c9/lib/aws_public_ips/cli.rb#L5-L39) some of these.

### Mitigation Types

#### No Public Subnets

TODO

#### No IGW

TODO

#### Condition Key

TODO

#### Condition Key + Resource Type Limits

TODO

### Mitigations Per Service

Service            | No Public Subnets | No IGW  | Condition Key | Condition Key + Resource Type Limits | Need To Ban
-------------------|-------------------|---------|---------------|--------------------------------------|------------
API Gateway        | N/A               | N/A     | Partial       | Yes                                  | No
Athena             | N/A               | N/A     | N/A           | N/A                                  | No
CloudFront         | ?                 | ?       | ?             | ?                                    | ?
DynamoDB           | N/A               | N/A     | N/A           | N/A                                  | No
ECS                | ?                 | ?       | ?             | ?                                    | ?
EC2                | Yes-ish           | Yes     | Partial       | N/A                                  | No
EKS                | Partial           | Partial | No            | N/A                                  | Yes
ElasticCache       | N/A               | N/A     | Partial       | Yes                                  | No
ELB v1 / v2        | Yes               | Yes     | No            | N/A                                  | No
EMR                | ?                 | ?       | ?             | ?                                    | ?
Global Accelerator | No                | Yes     | No            | No                                   | No
Lambda             | N/A               | N/A     | Yes           | N/A                                  | No
Lightsail          | ?                 | ?       | ?             | ?                                    | ?
Neptune            | ?                 | ?       | ?             | ?                                    | ?
RDS                | Yes-ish           | Yes     | No            | N/A                                  | No
Redshift           | Yes-ish           | Yes     | No            | N/A                                  | No
S3 / SNS / SQS     | N/A               | N/A     | N/A           | N/A                                  | No


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
