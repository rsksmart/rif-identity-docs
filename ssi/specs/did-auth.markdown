---
layout: default
---

## DID authentication - a challenge–response authentication model based on DIDs

A challenge–response authentication is a family of protocols in which one party presents a question ("challenge") and another party must provide a valid answer ("response") to be authenticated. {% include ref.html id="10" %}

This protocol is based on CHAP{% include ref.html id="11" %} authentication protocol, OAuth 2.0 Authorization Framework{% include ref.html id="13" %} and Decentralized Identifiers (DIDs) v1.0 {% include ref.html id="2" %}.

It is designed to:

- Authenticate a user by proving than he/she controls a specific did
- Provide an _access token_ to the user that can be reused during a certain amount of time - wallet systems usually request user action to sign messages, enabling reusing access token reduces the amount of signatures required and improves the user experience
- Allow the user to refresh the _access token_ once it expired - sessions will still be valid beyond the _access token_ expiration time
- Give the user control of the session by providing a logout endpoint that will mark the current session as closed

It uses HTTPS as the message transport layer.

Alice needs to access Bob's service, so Bob needs to authenticate Alice:

1. Alice sends `POST /request_auth { did }` to Bob, where `did` is Alice's DID
2. Bob creates a deterministic _challenge_ (see [How to calculate a deterministic challenge](#how-to-calculate-a-deterministic-challenge)) to send to Alice and responds with `{ challenge }`.
3. Alice receives Bob's `challenge` and creates a JWT {% include ref.html id="1" %} with the following payload:
```javascript
{
  iss: `${aliceDid}`,
  aud: `${bobServiceUrl}`,
  exp: `${now + 2 min}`,
  nbf: `${now}`,
  iat: `${now}`,
  challenge: `${receivedChallenge}`,
}
```
Alice signs it with her `did`'s private key and sends  `POST /auth { response: jwt }` 
4. Bob verifies JWT signature, checks if the received `challenge` is ok (by calculating the deterministic one again) and, if necessary, performs business logic over the `did`. If it is a valid user, it creates an _access token_ and a _refresh token_. 
  - The _access token_ is a JWT signed with the service private key. The JWT MUST have, at least, the following payload:
```javascript
{
  iss: `${bobDid}`,
  aud: `${bobServiceUrl}`,
  sub: `${aliceDid}`
  exp: `${now + 10 min}`, // should be shorter than 15 minutes
  nbf: `${now}`,
  iat: `${now}`,
  // extra information that could be useful for the use case such as user metadata
}
```
  - The _refresh token_ is an opaque string (could be a random one) that will be associated to user session data in the server. Long expiration.
5. Alice authenticates next HTTPS requests using the received _access token_. See [how to send access tokens](#acess-token).
6. If the _access token_ is not expired, Bob authorises the request, if not, he answers with an HTTP 401 with the expired token error message.
7. If HTTP 401, Alice sends `POST /refresh-token` to Bob with the `refresh token`. See [how to send refresh tokens](#refresh-token).
8. Bob validates the _refresh token_ and the current session status, if valid, issues new _access token_ (with same data but new expiration), invalidates the received _refresh token_ and issues a new one.
9. Alice authenticates next HTTPS requests using the received _access token_.

![did auth]({{ site.baseurl }}/assets/img/ssi/08_did_auth.png)

## Logout

It will invalidate the current `did` session, so next time `/refresh-token` is invoked, it will not generate a new _access token_.

1. Alices sends `POST /logout` with the current _access token_
2. If the _access token_ is valid, Bob marks the associated _refresh token_ as logged out.
3. Alice sends `POST /refresh-token` to Bob with the `refresh token`
4. Bob does not refresh the _access token_ because the session was closed before.

NOTE: The logout process does not invalidate the current _access token_, it will still be valid until it expires, that's why it matters to implement short validity periods for _access token_. The logout just prevents the _access token_ to be renewed.

## How to calculate a deterministic challenge

Calculate a deterministic challenge prevents the server to maintain a state of emitted challenges while those challenges are valid for a certain amount of time.

Example:

```javascript
const challengeExpirationTime = 5 // in minutes
const serverSecret = 'this is the server super secret'
const userDid = `did:ethr:rsk:0x0123456789abcdef`

const timestamp = Math.floor((Date.now() / (challengeExpirationTime * 60 * 1000))

const challenge = keccak256(`${userDid}-${serverSecret}-${timestamp}`)
```

By doing this, the `challenge` will be valid during maximum 5 minutes. Once the user sends the signed challenge back to the service, the server MUST perform the same calculation and the received challenge must be coincident with the result of that new calculation, if not, an invalid challenge response will be sent.

## How to send tokens

There are different options to send the _access token_ and the _refresh token_. Request headers, request body or cookies can be used depending on the case and the developer election, please find below the different descriptions.

NOTE: If you decide to use cookies, please make sure that your service is secure enough to prevent Cross Site Request Forgery ([CSRF](https://owasp.org/www-community/attacks/csrf)) atttacks.

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

When the client performs a `POST /auth`, the server must set the `authorization` cookie with the following attributes: `HttpOnly`, `Secure` and `SameSite=Strict` (this new attribute prevents CSRF, but it is not supported by all the browsers yet). See more information about cookies [here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)

For example:

```
Set-Cookie: authorization=my.access.token; Secure; HttpOnly; SameSite=Strict
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
Set-Cookie: refresh-token=theRefreshToken; Secure; HttpOnly; SameSite=Strict
```

Then, the client browser will send the cookie on every request, so when the client makes a `POST /refresh-token`, the server will replace the existing cookies (`authorization` and `refresh-token`) with new values.

## Implementations

- [`@rskmsart/express-did-auth`](../libraries/express-did-auth) - in progress

## Extensions

- Separate authentication server from resource server
- Allow the user to associate many dids to the same identity, so the business logic over `did1` and `did2` should be the same if both dids are associated.

## Open work

- Build a repository of W3C Verifiable Credentials JSON Schema definition
- This protocol can be abstracted from the transport layer