---
layout: default
---

## DID authentication - a challenge–response authentication model based on DIDs

A challenge–response authentication is a family of protocols in which one party presents a question ("challenge") and another party must provide a valid answer ("response") to be authenticated. {% include ref.html id="10" %}

This protocol is based on CHAP{% include ref.html id="11" %} authentication protocol, W3C Verifiable Credentials model {% include ref.html id="7" %} and [uPort Selective Disclosure Flow](https://developer.uport.me/flows/selectivedisclosure).

It is designed to:

- Let the application request the user to share specific information that can be verified - this information can be used in business logic to grant or deny access
- Allow the user to opt-in or out to the information - it is a user-centric protocol, the user decides wether to share the information or not
- Provide the user an access credential that can be reused - wallet systems usually request user action to sign messages, enabling reusing access credentials the amount of signatures required and improves the user experience

It uses HTTPS as the message transport layer.

Alice needs to access Bob's service, so Bob needs to authenticate Alice:

1. Alice sends `POST /request_auth { did }` to Bob, where `did` is Alice's DID
2. Bob creates a random challenge to send to Alice, stores the pair `did-challenge`, and responses with `{ challenge, sdreq }` were `sdreq` is the _Selective disclosure request_
3. Alice receives Bob's challenge and selective disclosure request, obtains information required by the sdr, creates a JWT {% include ref.html id="1" %} with payload `{ challenge, sdres }` signed with `did`'s private key, where `sdres` is the selective disclosure response, and sends  `POST /auth { response: jwt({ challenge, sdres }) }`
4. Bob verifies JWT signature, compares the challenge in the payload to the stored challenge, and performs business logic over the selective disclosure response. If is a valid user, creates and responses a Verifiable Credential that can be used by Alice to access the HTTPS service.
5. Alice accesses next HTTPS requests using header `Authentication` with the Credential received

The access credential issued by Bob can be reused until `expirationDate` value.

The authentication protocol can be modified to avoid using selective disclosure request. This is useful when the service does not need to request credentials to the user.

Also, it can be modified to user just a token, without reusable credentials. A simple implementation could require Alice to request a challenge for each interaction with the service. A smarter implementation would respond a new challenge after each interaction, Alice will need to make only one extra request to get the first challenge.

## Sequence diagram

![](/assets/img/ssi/08_did_auth.png)

### Open work

- Build a standard interface for selective disclosure requests and responses
- This protocol can be abstracted from the transport layer

### Implementations

- [`@rskmsart/rif-id-did-auth`](#)
- [RIF Data Vault authentication](/data-vault/architecture/auth)
