---
layout: default
---

## Applications

- [Issuer App](./issuer-app) - React application that handles the administrative tasks of issuing and rejecting credential requests - uses [issuer backend](../services) to handle requests and SSI operations
- [Holder App](./holder-app) - React Native wallet that holds and requests Verifiable Credentials - uses [user-centric data vault]({{ site.baseurl }}/data-vault) service to store backup of user's SSI properties
