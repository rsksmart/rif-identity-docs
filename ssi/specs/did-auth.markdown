---
layout: default
---

## DID authentication - a challenge–response authentication model based on DIDs

A challenge–response authentication is a family of protocols in which one party presents a question ("challenge") and another party must provide a valid answer ("response") to be authenticated. {% include ref.html id="10" %}

This protocol is based on CHAP{% include ref.html id="11" %} authentication protocol, W3C Verifiable Credentials model {% include ref.html id="7" %} and W3C Verifiable Credentials JSON Schema Specification {% include ref.html id="12" %}.

It is designed to:

- Let the application request the user to share specific information that can be verified - this information can be used in business logic to grant or deny access
- Allow the user to opt-in or out to the information - it is a user-centric protocol, the user decides wether to share the information or not
- Provide an access token to the user that can be reused - wallet systems usually request user action to sign messages, enabling reusing access token reduces the amount of signatures required and improves the user experience

It uses HTTPS as the message transport layer.

Alice needs to access Bob's service, so Bob needs to authenticate Alice:

1. Alice sends `POST /request_auth { did }` to Bob, where `did` is Alice's DID
2. Bob creates a random challenge to send to Alice, stores the pair `did-challenge`, and responses with `{ challenge, schemas }` were `schemas` is an array of published and well known names of _W3C Verifiable Credentials JSON Schema_. IE: `{ challenge: 'qwerty12345678', schemas: ['EmailCredentialSchema'] }`
3. Alice receives Bob's challenge and schemas request, obtains information required, creates a JWT {% include ref.html id="1" %} with payload `{ challenge, schemasResponse }` signed with `did`'s private key, where `schemasResponse` is the required schema response, and sends  `POST /auth { response: jwt({ challenge, schemaResponse }) }`
4. Bob verifies JWT signature, compares the challenge in the payload to the stored challenge, and performs business logic over the schemas response. If is a valid user, creates and responses an access token. That access token is a `jwt({ issuer, subject, expiration, audience, schemaResponse })`.
5. Alice accesses next HTTPS requests using header `Authentication` with the Credential received

The access credential issued by Bob can be reused until `expirationDate` value.

The schemas request is optional and it depends on the service needings. This is useful when the service does not need to request credentials to the user.

This protocol can be modified to use disposable tokens, without reusable credentials. A simple implementation may require Alice to request a new challenge for each interaction with the service. A smarter implementation would respond with a new challenge after each interaction, so Alice will need to save the next challenge to be used and the protocol will prompt _Alice_ to make only one extra request to get the first challenge.

## Sequence diagram

![did auth]({{ site.baseurl }}/assets/img/ssi/08_did_auth.png)

### Open work

- Build a repository of W3C Verifiable Credentials JSON Schema definition
- This protocol can be abstracted from the transport layer

### Implementations

- [`@rskmsart/rif-id-did-auth`](../libraries/express-did-auth) - in progress
<!-- - [RIF Data Vault authentication]({{ site.baseurl }}/data-vault/architecture/auth) -->
