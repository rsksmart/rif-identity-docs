---
layout: default
---

## DID authentication - a challenge–response authentication model based on DIDs

A challenge–response authentication is a family of protocols in which one party presents a question ("challenge") and another party must provide a valid answer ("response") to be authenticated. {% include ref.html id="10" %}

This protocol is based on CHAP{% include ref.html id="11" %} authentication protocol, OAuth 2.0 Authorization Framework{% include ref.html id="13" %}, W3C Verifiable Credentials model {% include ref.html id="7" %}, [uPort selective disclosure implementation](https://developer.uport.me/flows/selectivedisclosure) and W3C Verifiable Credentials JSON Schema Specification {% include ref.html id="12" %}.

It is designed to:

- Let the application request the user to share specific information that can be verified - this information can be used in business logic to grant or deny access
- Allow the user to opt-in or out to the information - it is a user-centric protocol, the user decides wether to share the information or not
- Provide an access token to the user that can be reused (with or without the needing of refreshing it with a _refresh token_) - wallet systems usually request user action to sign messages, enabling reusing access token reduces the amount of signatures required and improves the user experience

It uses HTTPS as the message transport layer.

Note: this protocol provides an optional flow that includes a refresh token, both flows are explained with the same example

Alice needs to access Bob's service, so Bob needs to authenticate Alice:

**With an access token**

1. Alice sends `POST /request_auth { did }` to Bob, where `did` is Alice's DID
2. Bob creates a random _challenge_ to send to Alice, stores the pair `did-challenge`, and responses with `{ challenge, sdr }` were `sdr` is a [selective disclosure request](#request)
3. Alice receives Bob's `challenge` and `sdr` request, obtains information required, creates a JWT {% include ref.html id="1" %} with payload `{ challenge, sdr }` signed with `did`'s private key, where `sdr` is the [selective disclosure response](#response), and sends  `POST /auth { response: jwt({ challenge, sdr }) }`
4. Bob verifies JWT signature, compares the _challenge_ in the payload to the stored _challenge_, and performs business logic over the sdr. If it is a valid user, creates and responds with an _access token_. That _access token_ is a `jwt({ issuer, subject, expiration, audience, sdr })`.
5. Alice authenticates next HTTPS requests using the received _access token_. See [how to send access tokens](#access-token).

The _access token_ issued by Bob can be reused until `expirationDate` value.

![did auth]({{ site.baseurl }}/assets/img/ssi/08_did_auth.png)

**With an access token AND a refresh token**

1. Alice sends `POST /request_auth { did }` to Bob, where `did` is Alice's DID
2. Bob creates a random _challenge_ to send to Alice, stores the pair `did-challenge`, and responses with `{ challenge, sdr }` were `sdr` is a [selective disclosure request](#request)
3. Alice receives Bob's `challenge` and `sdr` request, obtains information required, creates a JWT {% include ref.html id="1" %} with payload `{ challenge, sdr }` signed with `did`'s private key, where `sdr` is the [selective disclosure response](#response), and sends  `POST /auth { response: jwt({ challenge, sdr }) }`
4. Bob verifies JWT signature, compares the _challenge_ in the payload to the stored _challenge_, and performs business logic over the sdr. If it is a valid user, it creates an _access token_ and a _refresh token_. 
  - The _access token_ is a `jwt({ issuer, subject, expiration, audience, sdr })` and its expiration should be short (less than 10 minutes). 
  - The _refresh token_ is an opaque string that will be associated to user session data in the server (last-login, did, user-agent, revoked, etc). Long expiration.
5. Alice authenticates next HTTPS requests using the received _access token_. See [how to send access tokens](#acess-token).
6. If the _access token_ is not expired, Bob authorises the request, if not, he answers with an HTTP 401 with the expired token error message.
7. If HTTP 401, Alice sends `POST /refresh-token` to Bob with the `refresh token`. See [how to send refresh tokens](#refresh-token).
8. Bob validates the _refresh token_ and, if valid, issues new _access token_ (with same data but new expiration), invalidates the received _refresh token_ and issues a new one.
9. Alice authenticates next HTTPS requests using the received _access token_.

![did auth]({{ site.baseurl }}/assets/img/ssi/09_did_auth_refresh_token.png)

The selective disclosure request is optional and it depends on the service needings.
- Open apps: it needs just a proof that the user controls the did. In that case the `challenge` is enough
- Permissioned apps: it needs a proof that the user controls the did and also proofs that the user fullfil the business needings. IE: be older than 18 years old.

## Selective disclosure

It is strongly based on [uPort implementation](https://developer.uport.me/flows/selectivedisclosure), where the service requires certain information and the user responds with it.

The selective disclosure request must be compatible with [uPort DAF implementation](https://github.com/uport-project/daf/blob/d7714e5b3c2f00a90a861488deb2d37fba750173/packages/daf-selective-disclosure/src/action-handler.ts#L16-L23), so it must implement the following interfaces.


### Request 

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

`replyUrl`: the auth endpoint.

`claims`: needed claims that may be or not be part of a credential. IE: `preferredLanguage`.

`credentials`: array of W3C Verifiable Credentials JSON Schema names. Those schemas definitions will be published in an open repository in Github. IE: `EmailCredential`, `BirthdateCredential`.


### Response 

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

## How to send tokens

There are different options to send the _access token_ and the _refresh token_. Request headers, request body or request cookies can be used depending on the case and the developer election, please find below the different descriptions.

### Access token

#### Authorization Header

It must be placed in the `Authorization` header following the `DIDAuth` scheme. This scheme will be present in HTTP Authentication Scheme Registry{% include ref.html id="14" %}.

For example:

```
  GET /resource HTTP/1.1
  Host: server.example.com
  Authorization: DIDAuth my.access.token
```

#### Cookie

When the client performs a `POST /auth`, the server must set the `authorization` cookie with the following attributes: `HttpOnly`, `Secure` and `SameSite=Strict`. See more information about cookies [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)

For example:

```
Set-Cookie: authorization=my.access.token; Secure; HttpOnly
```

Then, the client browser will send the cookie on every request.

### Refresh token

#### Body

It must be placed in the body of the request as a `refresh-token` field.

For example:

```
  POST /refresh-token HTTP/1.1
  Host: server.example.com

  {
    refresh-token: 'theRefreshToken'
  }
```

#### Cookie

When the client performs a `POST /auth`, the server must set the `refresh-token` cookie with the following attributes: `HttpOnly`, `Secure` and `SameSite=Strict`. 

For example:

```
Set-Cookie: refresh-token=theRefreshToken; Secure; HttpOnly
```

Then, the client browser will send the cookie on every request, so when the client makes a `POST /refresh-token`, the server will replace the existing cookies (`authorization` and `refresh-token`) with new values.

## Implementations

- [`@rskmsart/rif-id-did-auth`](../libraries/express-did-auth) - in progress
<!-- - [RIF Data Vault authentication]({{ site.baseurl }}/data-vault/architecture/auth) -->

## Extensions

Separate authentication server from resource server

This protocol can be modified to use disposable tokens in order to ensure that the user is always controlling the did, without reusable credentials. A simple implementation may require Alice to request a new challenge for each interaction with the service. A smarter implementation would respond with a new challenge after each interaction, so Alice will need to save the next challenge to be used and the protocol will prompt _Alice_ to make only one extra request to get the first challenge. One of the cons of this approach is that the client will have to sign many times, it may be a limitation from the user experience perspective.

## Open work

- Build a repository of W3C Verifiable Credentials JSON Schema definition
- This protocol can be abstracted from the transport layer