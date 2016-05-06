---
title: "Solving Authentication in Microservices"
date: 2016-05-06 17:17:22 -0400
---

Microservices are all the rage these days in application architecture. Employed properly, they do a great many things to help combat the [inevitable complexities of large applications](https://www.oreilly.com/ideas/microservices-shift-complexity-to-where-it-belongs). They also introduce a [new suite of challenges](http://highscalability.com/blog/2014/4/8/microservices-not-a-free-lunch.html) to be addressed by the developer, though.

While working on a project recently, my team and I identified an opportunity to split a large portion of related functionality into a microservice. We would then have a main application, which included the HTML and UI, and a data API microservice which performed integration with a third party service.

## Requirements

Early on, the team identified a challenge that would have to be solved: the main application had knowledge about the user currently logged in and using the application, but the microservice did not. How, then, would we secure our microservice in such a way that authorized users could make API requests?

As we evaluated possible solutions, several requirements emerged.

### No shared access to the user database

Using a database as an "API"-like interface is generally a bad idea. The data structure of an application should be considered an implementation detail of the owning application. Allowing multiple applications to access the same data storage system creates tight coupling and makes data structure refactoring more difficult.

### Microservice must have access to the current user context

In order for the microservice to operate correctly, it does need to know about the current user, in order to properly scope data and enforce authorization.

### User data must not be redundant

We did not want to rely on a "synchronizer" mechanism to populate data from one database into another. This would always present latency issues and just another place to fail.

### Must support single sign in and out

Obviously making the user sign in to each microservice was not a viable option, but the extended requirement of single sign out is a particularly difficult challenge. Often SSO is implemented using a temporary token of sorts, but once the token is given to the user (browser), it must be considered "compromised", as there is no guarantee that the browser will discard it in a timely manner. And because of the same-origin-policy in browsers, simply sharing a cookie was impractical.

### Sign out must be expedient

Of all the requirements listed above, Single Sign Out presents perhaps the greatest challenge. Because of the sensitive nature of our data, a temporary token (such as OAuth) that automatically expires "in an hour" would not afford us enough protection. We felt that the overlapping window of time where a user had signed out, but potentially still held a token affording access to the protected data API was unacceptable. Signing out must _immediately_ revoke all access to protected resources.

## Solutions

We looked at a few existing solutions to this problem, but the only real contenders to emerge fell short of our requirements, as described below.

### OAuth

OAuth initially seemed like the most obvious choice, but it should be noted that OAuth is a solution geared toward _authorization_, not authentication. It says so on the [OAuth.net](http://oauth.net/articles/authentication/) website.

OAuth is additionally intended for scenarios where there are three parties: a user, an identity provider, and an identity consumer. While similar to our microservice example, the identity provider and consumer are more commonly third parties, for example, "Sign In with Facebook" on popular websites. Put another way, OAuth is primarily geared toward scenarios where one party would like to make requests to another party _on behalf of_ the user. This does not accurately describe our microservice model.

### SAML

[SAML](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language) would seem another likely candidate, as it specifically addresses the in-browser single sign-on domain. However, SAML is notoriously complicated, and for our application, felt very heavyweight.

SAML works by utilizing hidden HTML forms in the browser to courier the authentication assertions between the identity provider and service provider. For our application, this added extra layers of complexity, since the service provider would be an API, intended to be consumed by both an in-browser JavaScript Single Page Application as well as other back-end server applications.

### Javascript Web Token

[JWT](http://jwt.io) has a lot of promising features, chief among them being it's portability and independence from any back-end persistence layer. The token uses the client (browser) as a courier for an encoded payload that only the identity provider and service provider can _verify_.

However, because it lacked a central storage mechanism, it failed to solve the single-sign-off requirement. Undoubtedly, we could have applied some other tactics to invalidate the token, but by the time we had engineered our way around this problem, we felt we had lost much of the ground JWT gave us for free.

### LDAP or commercial products

There are a litany of commercial products touting "single sign on" capabilities, but most of these are stand-alone identify providers that everything else must then plug into. At the time of this endeavor, we had little interest in replacing our existing user authentication mechanism in the main application. Whatever solution we chose needed to plug in or bolt onto what we already had.

## ~~Not~~ Built Here

There are some problem domains in the software world that have been solved a thousand times, and I would never recommend a development shop rolling their own solution to the problem. Authentication is a prime example of this. Given the complexities and eccentricities of authentication in a browser, rolling your own authentication scheme all but guarantees _someone_ will discover a critical gap or vulnerability in your solution that can be exploited to great detriment.

Nevertheless, having been unsatisfied with the solutions above solutions, we decided to create our own token-based authentication scheme to address the requirements. Our solution would be heavily inspired by JWT, but would employ a centralized token repository to enable immediate invalidation of tokens when a user signs out.

## Stay Tuned

Hopefully, this has been an insightful article, detailing the complexities of federated authentication in a microservice infrastructure. This should also shed some light on why we chose to ignore conventional wisdom and hash out our own solution.

I will detail the architecture of our proposed solution in a future post, coming soon, so please check back for updates!
