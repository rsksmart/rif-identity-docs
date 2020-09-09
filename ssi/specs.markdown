---
layout: default
---

## Specs

RIF Self-sovereign Identity works under a set of JWT-based and blockchain-based protocols building a user centric verifiable credential model.

### What is a JWT?

JSON Web Token (JWT) is a compact, URL-safe means of representing claims to be transferred between two parties.<sup><a href="#ref-1">1</a></sup> These claims can be secured with digital signatures or Message Authentication Codes, and/or encrypted with different algorithms. The following is an example of a JWT Claims Set:

     {"iss":"joe",
      "exp":1300819380,
      "http://example.com/is_root":true}

## Identity representation

Decentralized identifiers (DIDs) are a new type of identifier that enables verifiable, decentralized digital identity.<sup><a href="#ref-2">2</a></sup> A DID identifies any subject such as a person, an organization, a thing, an abstract entity, and so on.

DIDs are URIs that represent the identities in a unique manner - the DID URI contains a public key representing the owner of that identity. A DID identity is mapped to a DID Document that expresses current state of that identity. Control delegations and identity ownership transfers are some of the actions that can be performed to modify the public state of a DID Document - to do so in a secure and public way RSK blockchain is used.

RIF Identity currently supports `rsk` and `rsk:testnet` Ethr-DID method<sup><a href="#ref-3">3</a></sup> networks. As an example, this two DIDs are valid RSK and RSK Testnet DIDs:

```
did:ethr:rsk:0x1fab9a0e24ffc209b01faa5a61ad4366982d0b7f
did:ethr:rsk:testnet:0x487ff2e63c8f89a97b6f92d184e2e80fdcdc6ee6
```

## Multi identity model

Imagine what would happen if each time you present your national identity card you reveal your blockchain address which can show the amount of funds you have and historical transaction information. To enable users keep their data private and safe, RIF Identity specifies a deterministic identity derivation standard schema that will allow users to obtain multiple public identities for free with the guarantee of recovery on any device.

This schema is based on BIP-32<sup><a href="#ref-4">4</a></sup> hierarchical deterministic derivations of public keys. With a seed you can create multiple private keys, that private keys control multiple identities. These identities, as far as the model is correctly designed, do not share information that can associate one with another.

![multi_identity_model](/assets/img/ssi/04_multi_identity_model.png)

## Verifiable credentials model

Credentials are a part of our daily lives; driver's licenses are used to assert that we are capable of operating a motor vehicle, university degrees can be used to assert our level of education, and government-issued passports enable us to travel between countries.<sup><a href="#ref-5">5</a></sup> 

If credentials are cryptographically signed, the holder of the credential does not need any action on the issuer to prove that credential was issued by it. In addition, the identity controller of the subject that that credential was issued to can provide a cryptographic proof expressing control of the identity. With this proof schema the verification for a credential presentation consists of proving two cryptographically signed messages.

![multi_identity_model](/assets/img/ssi/02_vc_model.png)

For example, this model enables a transport office to issue a credential for an approved driver (Alice), that is able to use that credential in a transport control. The officer who verifies the credential does not need access to any centralized database to attest the driving license was issued to the person presenting it.

![multi_identity_model](/assets/img/ssi/03_vc_model_application.png)

In general, the model is represented by three different entities:

- Holder
- Issuer
- Verifier

This three entities can perform 4 different basic actions:

- Issue credentials
- Verify credentials
- Present credentials
- Verify presentations

## References

1. <span id="ref-1"></span> [JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)
2. <span id="ref-2"></span> [Decentralized Identifiers (DIDs) v1.0](https://w3c.github.io/did-core/)
3. <span id="ref-3"></span> [ETHR DID Method Specification](https://github.com/decentralized-identity/ethr-did-resolver/blob/master/doc/did-method-spec.md)
4. <span id="ref-4"></span> [BIP-0032 - Hierarchical Deterministic Wallets](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)
5. <span id="ref-5"></span> [Verifiable Credentials Data Model 1.0](https://www.w3.org/TR/vc-data-model/)


