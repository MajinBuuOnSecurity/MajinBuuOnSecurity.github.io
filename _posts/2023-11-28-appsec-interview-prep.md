---
layout: post
toc: true
title: "[WIP] WebAppSec Interviewing Notes"
---

This is a post of interview preparation notes, for myself to refer in the future, ideally never though.

I haven't specialized in WebAppSec in years, and would much rather do cloud security in my next role, but the majority of positions that are both open and pay well are AppSec. :\|

Part of the reason for this is the hiring managers are also looking for someone who can add features / contribute to the product as a software engineer in these roles.

## Summary

I'll write one later.

Studying this stuff is like LeetCoding: I will likely not benefit long-term from it. For any real life coding challenge I can easily determine what I don't know, and learn it. If I am implementing mTLS auth I might look at [https://github.blog/2023-08-17-mtls-when-certificate-authentication-is-done-wrong/](https://github.blog/2023-08-17-mtls-when-certificate-authentication-is-done-wrong/) but otherwise I won't.

On the other hand, I will happily read [DDIA](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321/ref=sr_1_4) for the 5th time, as that knowledge will help me for life.


## Why I don't Like AppSec

James Kettle is my favorite AppSec person.

Why?

Offensive AppSec research talks on new and novel vulnerability classes practically every year.

That's just on offense, however.

Unfortunately, I don't see much undifferentiated defensive work from AppSec teams that can benefit all other companies -- which is what motivates me.

It is just a slog of repeating the same work everyone else does:
- Bug Bounty program
- Responding to suspected ATOs
- Vulnerability Management
- Refactoring your codebase to use the same patterns everyone else uses (CSP)
- Writing a custom version of [AuthTables](https://github.com/magoo/AuthTables)
- Writing a custom fuzzer / dynamic tester for IDORs
- Writing a 'revoke unused privileges' service
- Writing SemGrep rules, CodeQL, or whatever

Exceptions to this have been [Pyre](https://github.com/facebook/pyre-check) from Facebook. And I'm not sure what else.

That took a team of PhDs writing OCaml and only applies to typed Python codebases.

## Vulnerability Classes

Acronyms galour:
- CORS
- CSRF
- CSP
- XSS
- SOP
- SQLi / RCE
- SSRF

Some others:
- DAST 
- SAST 
- WAFs

Ones I want a refresher on:
- OAUTH
- OIDC
- Request Smuggling
- Desynchronization Attacks
- SSTI (Server Side Template Injection)
- SRI
- XSS via PostMessage

What I am hoping no interviewer asks me about:
- [Meltdown and Spectre](https://meltdownattack.com/)
- JSONP (How does this work again?)

And I'm not sure what else.

### CSP

Some resources from around \~2016:

- ["CSP Is Dead, Long Live CSP! On the Insecurity of Whitelists and the Future of Content Security Policy"](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/45542.pdf)

- ["GitHub’s CSP journey"](https://github.blog/2016-04-12-githubs-csp-journey/)
- ["Postcards from the post-XSS world (2011)"](https://lcamtuf.coredump.cx/postxss/)

- ["Code-Reuse Attacks for the Web: Breaking Cross-Site Scripting Mitigations via Script Gadgets"](https://acmccs.github.io/papers/p1709-lekiesA.pdf)

### CORS

...

### SOP

...

### Desynchronization Attacks / Request Smuggling 

...
