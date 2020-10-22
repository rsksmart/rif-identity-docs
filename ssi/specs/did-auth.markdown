---
layout: default
---

# DID authentication - a challenge–response authentication model based on DIDs

A challenge–response authentication is a family of protocols in which one party presents a question ("challenge") and another party must provide a valid answer ("response") to be authenticated. {% include ref.html id="10" %}

# Table of contents

- [State of the art](#state-of-the-art)
- [Motivation](#motivation)
- [Endpoints](#endpoints)
  - [Signup](#signup)
    - [Selective disclosure](#selective-disclosure)
  - [Login](#login)
  - [Logout](#logout)
- [Implementations](#implementations)
- [Extensions](#extensions)
- [Open work](#open-work)
- [Apendix](#appendix)
  - [How to calculate a deterministic challenge](#how-to-calculate-a-deterministic-challenge)
  - [How to send tokens](#how-to-send-tokens)

# State of the art

Nowadays, authentication is handled in a centralized way. Most applications delegate the process of authenticating users to third-party services ([OIDC](https://openid.net/connect/)), and the authenticity of that information relies on those services providers. The problem with this type of authentication is that the user's information is not controlled by her, the third-party services are those who control it. Today's decentralized applications cannot rely on this authentication, since they must prove that the user is the one who controls their information and not a third party, that's why a new protocol is required.

# Motivation

The motivation of this protocol is to provide a user centric authentication and registration mechanism to be used by backend services of the web 3.0. There are two main type of services: _open_ (public access) and _permissioned_ (the user must provide certain information to access it). This proposal allows an external service to prove that the user controls her did and, if necessary, it may ask the user to share some private information associated to that did in order to register the user in the service.
If it is a permissioned service, it may ask for verified user related information (ie: Verifiable Credentials{% include ref.html id="7" %}) and check if the shared information has been issued by reliable entities.

It is designed to:

- Register users by requesting them to share specific information that can be verified - this information can be used in business logic to grant or deny access
- Allow the user to opt-in or out to the information - it is a user-centric protocol, the user decides wether to share the information or not
- Authenticate a user by proving that she controls a specific did
- Provide an _access token_ to the user that can be reused during a certain amount of time - wallet systems usually request user action to sign messages, enabling reusing access token reduces the amount of signatures required and improves the user experience


This protocol is based on the following specifications:
- CHAP{% include ref.html id="11" %} authentication protocol
- OAuth 2.0 Authorization Framework{% include ref.html id="13" %}.
- Decentralized Identifiers (DIDs) v1.0 {% include ref.html id="2" %}
- W3C Verifiable Credentials model {% include ref.html id="7" %}
- [uPort selective disclosure implementation](https://developer.uport.me/flows/selectivedisclosure)
- W3C Verifiable Credentials JSON Schema Specification {% include ref.html id="12" %}

It uses HTTPS as the message transport layer.

# Endpoints

## Signup

This section is OPTIONAL, it depends on the service needings. Some services may not need to register users before letting them enter in.

1. _Client_ sends `POST /request-signup { did }` to _Service_, where `did` is _User_'s DID
2. _Service_ creates a deterministic _challenge_ (see [How to calculate a deterministic challenge](#how-to-calculate-a-deterministic-challenge)) to send to _Client_ and responds with `{ challenge, sdr }` were `sdr` is the [selective disclosure request](#request) defined by the _Service_. The `sdr` MUST be sent in a signed JWT format.
3. If `sdr`, _Client_ obtains the information required, and builds the [selective disclosure response](#response)
4. _Client_ builds a JWT with the following payload:
```javascript
{
  iss: `${userDid}`,
  aud: `${serviceUrl}`,
  exp: `${now + 2 min}`,
  nbf: `${now}`,
  iat: `${now}`,
  challenge: `${receivedChallenge}`,
  sdr: `${builtSdr}`
}
```
and prompts the _User_ to sign it with her `did`'s private key.
5. _Client_ sends  `POST /signup { response: jwt }` to _Service_
6. _Service_ verifies JWT signature, checks if the received `challenge` is ok (by calculating the deterministic one again), and performs business logic over the `sdr`. If it is a valid user, it logges her in by creating an _access token_ and a _refresh token_.
  - The _access token_ is a JWT signed with the service private key. The JWT MUST have, at least, the following payload:
```javascript
{
  iss: `${serviceDid}`,
  aud: `${serviceUrl}`,
  sub: `${userDid}`
  exp: `${now + 10 min}`, // should be shorter than 15 minutes
  nbf: `${now}`,
  iat: `${now}`,
  // extra information that could be useful for the use case such as user metadata
}
```
  - The _refresh token_ is an opaque string (could be a random one) that will be associated to user session data in the server. Long expiration.
7. _Client_ authenticates next HTTPS requests using the received _access token_. See [how to send access tokens](#acess-token).
8. If the _access token_ is not expired, _Service_ authorises the request, if not, it answers with an HTTP 401 with the expired token error message.
9. If HTTP 401, _Client_ sends `POST /refresh-token` to _Service_ with the `refresh token`. See [how to send refresh tokens](#refresh-token).
10. _Service_ validates the _refresh token_ and the current session status, if valid, issues new _access token_ (with same data but new expiration), invalidates the received _refresh token_ and issues a new one.
11. _Client_ authenticates next HTTPS requests using the received _access token_.
12. _Service_ authorises the request.

![did auth]({{ site.baseurl }}/assets/img/ssi/10_did_sign_up.png)

The selective disclosure request is optional and it depends on the service needings.
- Open apps: it needs just a proof that the user controls the did. In that case the `challenge` is enough
- Permissioned apps: it needs a proof that the user controls the did and also proofs that the user fullfil the business needings. IE: be older than 18 years old.

### Selective disclosure

It is strongly based on [uPort implementation](https://developer.uport.me/flows/selectivedisclosure), where the service requires certain information and the user responds with it.

The selective disclosure request must be compatible with [uPort DAF implementation](https://github.com/uport-project/daf/blob/d7714e5b3c2f00a90a861488deb2d37fba750173/packages/daf-selective-disclosure/src/action-handler.ts#L16-L23), so it must implement the following interfaces.

#### Request 

```
export interface Claim {
  claimType: string
  claimValue: string
  reason?: string
  essential?: boolean
}

export interface SelectiveDisclosureRequest {
  issuer: string
  subject: string
  replyUrl?: string
  claims?: Claim[]
  credentials?: string[]
}
```

`issuer`: the service did. REQUIRED

`subject`: the user did (the one received when requesting the challenge). REQUIRED

`replyUrl`: the sign up endpoint.

`claims`: needed claims that may or not be part of a credential. IE: `preferredLanguage`.

`credentials`: array of W3C Verifiable Credentials JSON Schema names. Those schemas definitions will be published in an open repository in Github. IE: `EmailCredential`, `BirthdateCredential`.


#### Response 

```
import { VerifiableCredential } from 'did-jwt-vc'

export interface SelectiveDisclosureResponse {
  issuer: string
  subject: string
  claims?: Claim[]
  credentials: VerifiableCredential[]
}
```

`issuer`: the user did. REQUIRED

`subject`: the service did. REQUIRED

`claims`: requested claims. IE: `{ claimType: 'preferredLanguage', claimValue: 'english' }`

`credentials`: array of W3C Verifiable Credentials that implements the requested JSON schemas


## Login

1. _Client_ sends `POST /request-auth { did }` to _Service_, where `did` is _User_'s DID
2. _Service_ creates a deterministic _challenge_ (see [How to calculate a deterministic challenge](#how-to-calculate-a-deterministic-challenge)) to send to _Client_ and responds with `{ challenge }`.
3. _Client_ receives _Service_'s `challenge` and creates a JWT{% include ref.html id="1" %} with the following payload:
```javascript
{
  iss: `${userDid}`,
  aud: `${serviceUrl}`,
  exp: `${now + 2 min}`,
  nbf: `${now}`,
  iat: `${now}`,
  challenge: `${receivedChallenge}`,
}
```
_Client_ prompts the _User_ to sign it with her `did`'s private key.
4. _Client_ sends  `POST /auth { response: jwt }` with the just signed JWT.
5. _Service_ verifies JWT signature, checks if the received `challenge` is ok (by calculating the deterministic one again) and, if necessary, performs business logic over the `did` and the information related to it saved by the _Service_. If it is a valid user, it creates an _access token_ and a _refresh token_. 
  - The _access token_ is a JWT signed with the service private key. The JWT MUST have, at least, the following payload:
```javascript
{
  iss: `${serviceDid}`,
  aud: `${serviceUrl}`,
  sub: `${userDid}`
  exp: `${now + 10 min}`, // should be shorter than 15 minutes
  nbf: `${now}`,
  iat: `${now}`,
  // extra information that could be useful for the use case such as user metadata
}
```
  - The _refresh token_ is an opaque string (could be a random one) that will be associated to user session data in the server. Long expiration.
6. _Client_ authenticates next HTTPS requests using the received _access token_. See [how to send access tokens](#acess-token).
7. If the _access token_ is not expired, _Service_ authorises the request, if not, it answers with an HTTP 401 with the expired token error message.
8. If HTTP 401, _Client_ sends `POST /refresh-token` to _Service_ with the `refresh token`. See [how to send refresh tokens](#refresh-token).
9. _Service_ validates the _refresh token_ and the current session status, if valid, issues new _access token_ (with same data but new expiration), invalidates the received _refresh token_ and issues a new one.
10. _Client_ authenticates next HTTPS requests using the just received _access token_.
11. _Service_ authorises the request.

![did auth]({{ site.baseurl }}/assets/img/ssi/08_did_auth.png)

## Logout

It will invalidate the current `did` session, so next time `/refresh-token` is invoked, it will not generate a new _access token_.

1. _Client_s sends `POST /logout` with the current _access token_
2. If the _access token_ is valid, _Service_ marks the associated _refresh token_ as logged out.
3. _Client_ sends `POST /refresh-token` to _Service_ with the `refresh token`
4. _Service_ does not refresh the _access token_ because the session was closed before.

NOTE: The logout process does not invalidate the current _access token_, it will still be valid until it expires, that's why it matters to implement short validity periods for _access token_. The logout just prevents the _access token_ to be renewed.

# Implementations

- [`@rskmsart/express-did-auth`](../libraries/express-did-auth) - in progress

# Extensions

- Separate authentication server from resource server. The goal is to differentiate the server that keeps the session data from the one that has the resources (API), so this last server could be stateless. By doing so, the user authenticates and refresh tokens with a Security Token Service that emits tokens can be used in another services that owns the business resources (ie: `/products`)
- Allow the user to express control of different identities in a private manner. Lets the user use different identities for a single registration

# Open work

- Build a repository of W3C Verifiable Credentials JSON Schema definition
- This protocol can be abstracted from the transport layer

# Appendix

## How to calculate a deterministic challenge

Calculate a deterministic challenge prevents the server to maintain a state of emitted challenges while those challenges are valid for a certain amount of time.

Example:

```
const challengeExpirationTime = 5min
const serverSecret = 'this is the server super secret'
const userDid = `did:ethr:rsk:0x0123456789abcdef`

const timestamp = int(now / challengeExpirationTime)

const challenge = keccak256(userDid-serverSecret-timestamp`)
```

By doing this, the `challenge` will be valid for the next `challengeExpirationTime - now % challengeExpirationTime`. Once the user sends the signed challenge back to the service, the server MUST perform the same calculation and the received challenge must be coincident with the result of that new calculation, if not, an invalid challenge response will be sent.

## How to send tokens

There are different options to send the _access token_ and the _refresh token_. Request headers, request body or cookies can be used depending on the case and the developer election, please find below the different descriptions.

NOTE: If you decide to use cookies, please make sure that your service is secure enough to prevent Cross Site Request Forgery ([CSRF](https://owasp.org/www-community/attacks/csrf)) atttacks.

#### Access token

##### Authorization Header

It must be placed in the `Authorization` header following the `DIDAuth` scheme. This scheme will be present in HTTP Authentication Scheme Registry{% include ref.html id="14" %}.

For example:

```
  GET /resource HTTP/1.1
  Host: server.example.com
  Authorization: DIDAuth my.access.token
```

##### Cookie

When the client performs a `POST /auth`, the server must set the `authorization` cookie with the following attributes: `HttpOnly`, `Secure` and `SameSite=Strict` (this new attribute prevents CSRF, but it is not supported by all the browsers yet). See more information about cookies [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)

For example:

```
Set-Cookie: authorization=my.access.token; Secure; HttpOnly; SameSite=Strict
```

Then, the client browser will send the cookie on every request.

#### Refresh token

##### Body

It must be placed in the body of the request as a `refreshToken` field.

For example:

```
  POST /refresh-token HTTP/1.1
  Host: server.example.com

  {
    refreshToken: 'theRefreshToken'
  }
```

##### Cookie

When the client performs a `POST /auth`, the server must set the `refresh-token` cookie with the following attributes: `HttpOnly`, `Secure` and `SameSite=Strict`. 

For example:

```
Set-Cookie: refresh-token=theRefreshToken; Secure; HttpOnly; SameSite=Strict
```

Then, the client browser will send the cookie on every request, so when the client makes a `POST /refresh-token`, the server will replace the existing cookies (`authorization` and `refresh-token`) with new values.
