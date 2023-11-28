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
- Refactoring your codebase to use the same patterns and HTTP headers everyone else uses (CSP)
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
- HSTS
- SOP
- SQLi / RCE
- SSRF
- XFO
- XSS

Some others:
- DAST 
- SAST 
- WAFs

Ones I want a refresher on:
- CT
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
- Esoteric trivia from [Ivan Ristic's book](https://www.amazon.com/Bulletproof-TLS-PKI-Second-Understanding/dp/1907117091/ref=sr_1_1)

And I'm not sure what else.

### The Very Basics

dguido NYU Poly course from Fall 2010:
- [Web Hacking 1](https://vimeo.com/21424320)
- [Web Hacking 2](https://vimeo.com/32550671)

Others:
- [https://trailofbits.github.io/ctf/web/exploits.html](https://trailofbits.github.io/ctf/web/exploits.html)
- [WAHH](https://www.google.com/search?q=filetype%3Apdf+web+hackers+handbook&oq=filetype%3Apdf+web+hackers+handbook)
- [BHH](https://www.google.com/search?q=filetype%3Apdf+browser+hackers+handbook)

### Auth Tokens

S2S
- Ptacek post, Latacora, 2018 [https://www.latacora.com/blog/2018/06/12/a-childs-garden/](https://www.latacora.com/blog/2018/06/12/a-childs-garden/)

Favorite quotes:
> Most API keys are bearer tokens. OAuth is an elaborate scheme for generating and relaying bearer tokens. SAML assertions are delivered in bearer tokens.

>From an application security perspective, “do the simplest thing you can get away with” has a lot of merit.


### CSP

Some resources from around \~2016:

- ["CSP Is Dead, Long Live CSP! On the Insecurity of Whitelists and the Future of Content Security Policy"](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/45542.pdf)

- ["GitHub’s CSP journey"](https://github.blog/2016-04-12-githubs-csp-journey/)
- ["Postcards from the post-XSS world (2011)"](https://lcamtuf.coredump.cx/postxss/)

- ["Code-Reuse Attacks for the Web: Breaking Cross-Site Scripting Mitigations via Script Gadgets"](https://acmccs.github.io/papers/p1709-lekiesA.pdf)

### CORS

Popular request headers:

- Origin

Popular response headers:
- Access-Control-Allow-Origin
- Access-Control-Allow-Credentials

Rarer are these:
- Access-Control-Request-Method / Access-Control-Allow-Methods
- Access-Control-Request-Headers / Access-Control-Allow-Headers

### JSONP

> JSONP payloads consist of an internal JSON payload wrapped by a pre-defined function call. When the script resource is loaded by the browser, the designated callback function will be invoked to process the wrapped JSON payload.


### What is cross-protocol exploitation?

Hmm



### SOP

XSS is heavily related.

Origin is a tuple of (protocol, hostname, port)

[https://en.wikipedia.org/wiki/Same-origin_policy](https://en.wikipedia.org/wiki/Same-origin_policy)

"X.com can no way interact with any page elements on Y.com"

"JavaScript executing in the context of 1 document should not be allowed to access the context of another document."

"SOP check happens in the browser."

...


### Desynchronization Attacks / Request Smuggling 

...


## ChatGPT

Storing a lot of session token information client-side versus server-side involves tradeoffs regarding security, performance, and scalability:

### Storing Session Token Information Client-Side:

#### Pros:
1. **Reduced Server Load:** Offloading data storage to the client-side reduces server-side processing and storage requirements.
2. **Faster Access:** Client-side storage (e.g., cookies, local storage) allows for quicker access to session-related data without frequent server requests.
3. **Enhanced Responsiveness:** Clients can retrieve and manipulate session information locally, leading to faster user interactions.

#### Cons:
1. **Security Risks:** Client-side storage is vulnerable to various attacks (XSS, data tampering) if not properly secured, exposing sensitive session data.
2. **Data Integrity Concerns:** Users might manipulate or tamper with client-side data, compromising the integrity of the session.
3. **Storage Limits:** Limited storage capacity (e.g., browser storage limits) might restrict the amount of data that can be stored client-side.

### Storing Session Token Information Server-Side:

#### Pros:
1. **Enhanced Security:** Centralized server-side storage reduces the risk of data exposure and manipulation compared to client-side storage.
2. **Better Control:** Allows for stricter access controls, audit trails, and encryption measures to protect session data.
3. **Scalability:** Easier to scale server-side storage solutions to accommodate larger amounts of session data.

#### Cons:
1. **Increased Server Load:** Storing and managing session data server-side can increase server load, especially with a large number of active sessions.
2. **Network Overhead:** Frequent client-server communication for session data retrieval might introduce network latency and affect performance.
3. **Potential Bottlenecks:** Heavy reliance on server-side storage might create performance bottlenecks, particularly in distributed systems or under high load.

### Considerations:

- **Sensitive Data:** If the session data contains highly sensitive information, storing it server-side is often the safer choice.
- **Performance Requirements:** If responsiveness and speed are critical, selectively storing necessary session data client-side can enhance user experience.
- **Scalability Needs:** Consider the scalability of server-side storage solutions to accommodate potential growth in session data volume.

In practice, a hybrid approach is often used, storing essential session details (such as tokens, identifiers) on the client-side for quick access, while storing more sensitive or extensive session data, logs, or user-related details on the server-side for security and scalability. This approach balances performance, security, and scalability concerns effectively.

Storing session tokens in JavaScript versus cookies involves different tradeoffs related to security, accessibility, and ease of implementation:

### Storing Session Tokens in JavaScript:

#### Pros:
1. **Control:** Gives more control to developers on how and where the token is stored.
2. **Flexibility:** Can use various storage mechanisms like localStorage, sessionStorage, or custom methods.
3. **Reduced Vulnerability:** Tokens stored in JavaScript might be less susceptible to certain types of attacks like CSRF (Cross-Site Request Forgery).

#### Cons:
1. **Security Risks:** Vulnerable to XSS (Cross-Site Scripting) attacks; if an attacker gains access to JavaScript, they can easily steal the token.
2. **Limited Accessibility:** Might not be accessible across different tabs or windows due to the scope of JavaScript storage.
3. **Complexity:** Requires extra effort to manage and secure the token properly.

### Storing Session Tokens in Cookies:

#### Pros:
1. **Automatic Inclusion:** Cookies are automatically sent with every HTTP request to the associated domain, simplifying server-side implementation.
2. **Cross-Tab Accessibility:** Cookies are accessible across tabs and windows, enhancing user experience.
3. **Security (with precautions):** Can be made secure by using HTTPOnly and Secure flags to prevent access via JavaScript and restrict to secure connections only.

#### Cons:
1. **Vulnerability to CSRF:** Cookies are vulnerable to CSRF attacks if not properly protected with measures like CSRF tokens.
2. **Size Limitations:** Limited in size (usually around 4KB), restricting the amount of data that can be stored.
3. **Privacy Concerns:** Users might disable or clear cookies, affecting session persistence and causing inconvenience.

### Considerations:

- **Security Priority:** If security is paramount, cookies with appropriate flags (HTTPOnly, Secure) might be more suitable.
- **Usability and Access:** If you need cross-tab accessibility and ease of use, cookies are more convenient.
- **Development Complexity:** If you have specific storage requirements or want more control, JavaScript-based storage might be preferable.

In practice, a combination of both methods might be used, leveraging cookies for the session token and JavaScript for additional client-side data storage if needed, considering the tradeoffs to strike a balance between security, accessibility, and usability.

In other words,

> It is often said that cookies are safer against XSS (with the HttpOnly flag set), but using cookies makes your app more vulnerable to CSRF. Tokens within JS, on the other hand, are easier to exfiltrate through XSS but CSRF therefore becomes much less of a problem.

So you can use both. In a similar same way that a Pepper is used.
