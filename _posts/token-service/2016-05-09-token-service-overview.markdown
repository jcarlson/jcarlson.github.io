---
title: "Token Service: An Overview"
date: 2016-05-09 18:39:00 -0400
---

In a [previous post]({% post_url 2016-05-06-authentication-in-microservices %}), I discussed the challenges associated with federated authentication in microservices.

In this post, I'm going to talk about a proposed solution that myself and my team came up with to address the problem. Our pain points weren't quite resolved by the more popular off-the-shelf solutions available at the time, and we resorted to rolling our own. I would recommend giving that post a read for [more details on why we chose to re-invent the authentication piece]({% post_url 2016-05-06-authentication-in-microservices %}).

## Overview

The solution we came up with is similar to a two-legged OAuth implementation, in that a single _token_ will be used as authentication.

At a high level, the client exchanges a username/password for a temporary token, and the issuance of the token is recorded in a repository. The client may then present this token to other microservices "as good as" their username and password. The microservice can then verify the token's existence in the repository, as well as retrieve details of the token holder stored with the token in the repository, such as the user's name, email, and so on.

Although we say the token is "as good as" a username and password, there is no requirement that username and password be the means of authenticating a user. In fact, the Token Service implementation _doesn't actually specify how a user is authenticated_. In effect, we make an assumption that some authentication mechanism _already exists_, and the token is merely another protected resource which the user must obtain in order to access other protected resources.

### Terminology

To simplify the detailed explanation, let's talk about the key players in this architecture.

#### Token Authority

The Token Authority is tasked with allocating tokens to authenticated users who have otherwise already proved their identity, such as through HTTP Basic authentication, a session cookie, or another means of authentication. The Token Authority additionally manages the record keeping for allocated tokens. Active tokens are stored in the repository and expired tokens are purged from the repository. The Token Authority therefore requires read/write access to our Token Repository.

#### Token Client

The Token Client is the microservice which serves protected resources. The user must present a token to gain access to the protected resource, and the Token Client enforces this contract by validating the token against the Token Repository. The Token Client therefore requires read-only access to our Token Repository.

#### Token Consumer

The Token Consumer is ultimately the user or user agent. The Token Consumer requests a token from the Token Authority and presents it to the Token Client in order to gain access to protected resources. Importantly, Token Consumers do not necessarily need to represent a user. They can represent any business object; for example, the Token Consumer may be another application in the ecosystem.

## Token Flow

The following UML describes loosely what happens during a request for a protected resource.

{% include token-service-overview.svg %}

1. User agent (token consumer) requests a token from the Token Authority
2. Token authority verifies existing authentication (such as session cookie or HTTP Basic) and allocates a token
3. Token is recorded in the token repository
4. Token is returned to the user agent for use in future requests
5. User agent requests protected resource from Token Client application
6. Token Client verifies token against Token repository
7. Token Repository confirms token is active and valid
8. Protected resource is returned to the user agent

## Recap

As may already be evident, the flow of a token in this solution is strikingly similar to the way two-legged OAuth authentication works, and this is by design. Where we will start to see differences, however, is in the structure and anatomy of a token, which I will discuss in my next post. Stay tuned!
