---
layout: post
toc: true
title: "[Rough Draft] Write Notes: Preventing Accidental Internet-Exposure of AWS Resources"
---

[The Minto Pyramid Principle Textbook](https://www.barbaraminto.com/textbook.html) is helpful in writing. I figured I would write a post on how I wrote the other posts, mostly for myself in order to re-read this in the future and track differences in my writing approach over time.

## Building a Pyramid

The first few versions of the following posts were mostly me textually vomitting what I wanted to say, where I wanted to gather folks thoughts on the substance of my writing rather than the structure. I then refactored the posts to improve their structure, much like productionizing a Python script. E.g. Originally it was 1 big post -- but then I refactored it into 3 parts, since if I was a security engineer sending this post to a colleague, I'd want 1 post per question and answer pair.

It would be better to start with structure, and then write what I had to say. Granted, the more I wrote, the more I discovered challenges and areas I needed to research more in-depth. So I would have needed to iterate either way.

Minto says, in Chapter 3, you can build the Pyramid from the top down, or bottom up.

### Bottom Up

1. List all the points I want to make
2. Work out the relationships between them
3. Draw conclusions
4. Work backwards to get the introduction

### Top Down

1. Identify the subject
2. Decide the question
3. Give the answer
4. Check the S and C will lead to the question
5. Verify the Answer
6. Move to fill in the Key Line

## Part 1: VPC

### List of Points

At the highest level of abstraction, the points I want to make are:

- This post is just for services in a VPC
- 1000 ft. view image shows What Good Looks Like
- Banning IGWs solves Internet exposures (Maybe the Answer?)
- Ingress/Egress is tightly coupled (A complication.)
- How do we support Egress use-case? (A question.)
- We have 4 different options (Sort of an answer too.)

For each of the various options, I have a point or two I would like to highlight, to shed more light on the option.

### Relationships between Points

Minto says in Chapter 8, you can use a Problem Definition Framework in which R1 is "What don't we like about the current situation?" and R2 is "What we want instead?".

Once we have R1 and R2, the problem is defined. We look for the solution.

For "Part 1: VPC", we require 2 layers:
![alt text](https://i.imgur.com/AbGHXHo.png)

Minto says to move from left to right, and down. Hence the numbered purple arrows.

### Conclusions Drawn
### Introduction Options



After describing What Good Looks Like (arrow #1)

#### Option A

I could ignore Minto's advice, and go straight into:

> The reason this is complicated to implement, is because in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

> The reason they are tightly coupled: is an Internet Gateway (IGW) is necessary for both. So if an engineer has IAM permissions to create an IGW, or is unrestrained in how they create resources in a VPC that has one, they can expose resources to the Internet.

In this way, I combine both layers of the problem-definition into one complication.

Otherwise, I would need to give the first solution (banning IGWs) in the introduction.

Then I can have the next section titled Preventing by Design and start it with:

> Since an IGW is the root of all of our problems, we simply can ban their creation via SCP ("ec2:CreateInternetGateway") upon vending an account, (alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.)

Full reproduction:

> The reason this is complicated to implement, is because in AWS: Egress to the Internet is tightly coupled with Ingress from the Internet. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

> The reason they are tightly coupled: is an Internet Gateway (IGW) is necessary for both. So if an engineer has IAM permissions to create an IGW, or is unrestrained in how they create resources in a VPC that has one, they can expose resources to the Internet.

> The Egress use-case typical looks like:
> ![alt text](https://i.imgur.com/vKsdNOh.png)

> Whereas an accidental Internet-exposure might look like:
> ![alt text](https://i.imgur.com/1e4M8z4.gif)

> Or:       
> ![alt text](https://i.imgur.com/gyXZz2E.gif)

> There are many ways to expose resources to the Internet like this, but the key insight -- for VPCs at least -- is that they all require an Internet Gateway (IGW).

> ## Preventing by Design

> Since an IGW is the root of all of our problems, we simply can ban their creation via SCP 
(`"ec2:CreateInternetGateway"`) upon vending an account, (alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.)

> That's it. Problem solved! You can hand the subaccount over to the customer, they will never be able to make public-facing load balancers or EC2 instances regardless of their IAM permissions.

> You can then, put accounts that were vended this way with the IGW-deny-SCP, in an OU and achieve an AWS organization structure similar to [What Good Looks Like](#about-the-problem) above.

> But what if you need to support the Egress use-case above?

> Then you need to ensure your network architecture tightly couples a NAT Gateway with an Internet Gateway, by e.g. giving subaccounts a paved path to a NAT Gateway in another account, you can do this via:

#### Option B

I could follow Minto's advice, and go straight into:

> You can implement this by banning `"ec2:CreateInternetGateway"` in subaccounts via SCP.[^2111] 

> There are many ways an accidental Internet-exposure might take place, e.g.
> ![alt text](https://i.imgur.com/1e4M8z4.gif)

> Or:       
> ![alt text](https://i.imgur.com/gyXZz2E.gif)

>  But the key insight -- for VPCs at least -- is that they all require an Internet Gateway (IGW).

[^2111]: Alongside deleting all the IGWs/VPCs that AWS makes by default in new accounts.

> You can then hand subaccounts over to the customers and they will never be able to make public-facing load balancers or EC2 instances regardless of their IAM permissions.

> There is only one complication with this.

> In AWS: <ins>Egress to the Internet is tightly coupled with Ingress from the Internet</ins>. In most cases, only the former is required (for example, downloading libraries, patches, or OS updates).

> The reason they are tightly coupled: is an Internet Gateway (IGW) is necessary for both.

> The Egress use-case typical looks like:
> ![alt text](https://i.imgur.com/vKsdNOh.png)

> ## Supporting Egress without an IGW

> To support the Egress use-case you need to ensure your network architecture tightly couples a NAT Gateway with an Internet Gateway, by e.g. giving subaccounts a paved path to a NAT Gateway in another account, you can do this via:


### Picking an Option

Pros for A: 
- No Solutions in the Introduction
- Solely paints a picture, and is About The Problem

Pros for B:
- The solution for the first layer of the problem definition is right next to What Good Looks Like
- Less words
- The next section is more granular
- The first section isn't soley About The Problem IMO.
- More upfront


## Other Notes

I started to write `Preventing Accidental Internet-Exposure of AWS Resources (Part 2: Handling All Services)` however, I need to first explain a service-allowlist, so I stopped and wrote a self-contained post about new account baselines.

