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

## Part 1: EC2

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
### Conclusions Drawn
### Resultant Introduction


