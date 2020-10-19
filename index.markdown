---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---

# RIF Identity

RIF Identity is the identity and reputation layer within the RIF ecosystem. It is meant to allow users to easily control their IDs to interact in decentralized economies while building a self-sovereign reputation. This will enable people, especially those excluded from the traditional financial system, to participate in the decentralized digital economy of the future.

The main goal of RIF Identity is to protect users’ personal data, empowering them to manage who can access it and giving them full control of their reputation so they can use it to interact with multiple marketplaces and platforms with freedom to move from one to another without losing their track record, contacts and social value.

## Table of contents

- [Self-sovereign identity](./ssi)
    - [Specs](./ssi/specs)
    - [Architecture](./ssi/architecture)
    - [Libraries](./ssi/libraries)
        - `@rsksmart/rif-id-mnemonic`
        - `@rsksmart/rif-id-ethr-did`
        - `@rsksmart/rif-id-daf`
        - `@rsksmart/rif-id-core`
        - `@rsksmart/express-did-auth`
        - `@rsksmart/rif-node-utils`
    - [Services](./ssi/services)
        - Convey service - public transport layer for JWTs using IPFS <!-- TODO: THERE IS A LINK -->
        - Issuer service - serves for an application that allows receiving credential issuance requests and approving them manually
    - [Applications](./ssi/applications)
        - Issuer app - application that serves as the credential request manager. It allows to grant-deny requests or revoke existing credentials  <!-- TODO: THERE IS A LINK -->
        - Holder app - wallet used to store declarative details and credentials of it’s users  <!-- TODO: THERE IS A LINK -->
        - Verifier app - QR scanner app that verifies Verifiable Presentations
    - [FAQ](ssi/faq)
- [Data Vault](./data-vault)
    - Centralized Data Vault provider - an IPFS Data Vault provider
    - Data Vault service - a Data vault first approach. This service uses an IPFS node to pin files
    - Web Client SDK - a lightweight web client for the Data Vault service
- rLogin - a web tool that combines Web3 and W3C standard protocols to manage user's identity
- RIF Identity manager - a platform where users can manage their personal information and other components that make up their identity

## Repos

- [Documentation](https://github.com/rsksmart/rif-identity-docs)
- [SSI Javascript monorepo](https://github.com/rsksmart/rif-identity.js)
- [Node.js Services](https://github.com/rsksmart/rif-identity-services)
- [React.js and React Native apps](https://github.com/rsksmart/rif-identity-ui)
- [Data vault Javascript monorepo](https://github.com/rsksmart/rif-data-vault)
- [Ethr DID + RSK support](https://github.com/rsksmart/ethr-did)
- [Ethr DID dev utils](https://github.com/rsksmart/ethr-did-utils)
- [rLogin](https://github.com/rsksmart/rLogin)
- [RIF Identity manager](https://github.com/rsksmart/rif-identity-manager)
