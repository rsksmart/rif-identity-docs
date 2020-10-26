---
layout: default
---

## Express DID Auth - a challenge-response authentication model based in DIDs

This package is an implementation of the [DID Auth protocol](../specs/did-auth). It is designed to be easely integrated to any Express applications.

Main features:

- Automatically set up the needed endpoints to fullfil the full protocol specification (signup, auth, refresh token and logout)
- Allows to decide where to send the tokens (cookies, `Authorization` header or request body for _refresh token_)
- Extensibility: it allows to add any specific business logic over the authentication/signup methods
- Deterministic challenge generation
- Limit requests per did per timeslot

### Usage

#### Install
```
npm i @rsksmart/express-did-auth
```

#### Plug and play

This is the simplest approach. Just need to provide an express `app` and the desired configuration for the package and it will create the needed endpoints on your behalf.
They are:
- `GET /request-signup/:did`: `{ challenge, sdr? }`
- `POST /signup { response }`: `{ accessToken, refreshToken }` *if using cookies, will return just an HTTP 200 because the tokens will be set server side
- `GET /request-auth/:did`: `{ challenge }`
- `POST /auth { response }`: `{ accessToken, refreshToken }` *if using cookies, will return just an HTTP 200 because the tokens will be set server side
- `POST /refresh-token { refreshToken }`: `{ accessToken, refreshToken }` *if using cookies, will return just an HTTP 200 because the tokens will be set server side
- `POST /logout`: `{}`

```typescript
import express from 'express'
import setupApp, { ExpressDidAuthConfig } from '@rsksmart/express-did-auth'

const config: ExpressDidAuthConfig = {
  // your config
}

const app = express()

app.get('/not-protected', function (req, res) {
  res.send('This endpoint is not authenticating')
})

setupApp(config)(app)

app.get('/protected', function (req, res) {
  res.send('This endpoint is authenticating')
})

const port = process.env.PORT || 5000

app.listen(port, () => logger.info(`My express API with did-auth running in ${port}`))
```

#### Configure

All the configuration should be placed in just one object of type `ExpressDidAuthConfig`. That object may contain the following fields:

_REQUIRED_

`challengeSecret: string`: the secret that will be used to generate the deterministic challenge. See [how we create deterministic challenges](../specs/did-auth#how-to-calculate-a-deterministic-challenge)

`serviceUrl: string`: will be used as the [`audience`](https://tools.ietf.org/html/rfc7519#section-4.1.3) of all the JWTs expected or emitted by this package. Should be a URI that identifies your service in the context where it is run

`serviceDid: string`: the did controlled by the servie. Will be used to sign JWTs.

`serviceSigner: Signer`: the signing function associated to the `serviceDid`. MUST implement `ES256K` algorithm, please find an example [here](https://github.com/decentralized-identity/did-jwt/blob/c91d38cdd06635b418250048e329c509eab1e6d6/docs/guides/index.md#simplesigner)

_OPTIONAL_

`requestSignupPath: string`: the request signup endpoint route. _Default: `/request-signup`_

`signupPath: string`: the signup endpoint route. _Default: `/signup`_

`requestAuthPath: string`: the request auth endpoint route. _Default: `/request-auth`_

`authPath: string`: the auth endpoint route. _Default: `/auth`_

`logoutPath: string`: the logout endpoint route. _Default: `/logout`_

`refreshTokenPath: string`: the refresh token endpoint route. _Default: `/refresh-token`_

`challengeExpirationTimeInSeconds: number`: the max expiration time for the generated challenge when requesting signup or auth. MUST be provided in `seconds`. _Default: `300` (5 minutes)_

`maxRequestsPerTimeSlot: number`: the max amount of requests per did per timeslot. _Default: `20`_

`timeSlotInSeconds: number`: the amount of `seconds` that need to elapse before resetting the request counter. _Default: `600` (10 minutes)_

`userSessionDurationInHours: number`: the validity of each _refresh token_ in hours. _Default: `168` (one week)_

`rpcUrl: string`: rpc url used to [resolve](https://github.com/decentralized-identity/ethr-did-resolver) Ethr DID identities. _If not provided, will resolve using both RSK Mainnet and Testnet networks._

`networkName: string`: network name used to [resolve](https://github.com/decentralized-identity/ethr-did-resolver) Ethr DID identities. _If not provided, will resolve using both RSK Mainnet and Testnet networks._

`registry: string`: [DID Registry](https://github.com/uport-project/ethr-did-registry) address used to resolve Ethr DID identities. _Default: `0xdca7ef03e98e0dc2b855be647c39abe984fcf21b`_

`useCookies: boolean`: determines if the _access token_ and _refresh token_ are saved in cookies or are returned in the body of the response. If `true`, the tokens will be extracted from the cookies. See [how to send tokens](../specs/did-auth#how-to-send-tokens) for more information.

`accessTokenExpirationTimeInSeconds: number`: the validity in `seconds` of each _access token_. Remember that it should be short because the long validity is for the _refresh token_. _Default: `600` (10 minutes)_

`authenticationBusinessLogic: AuthenticationBusinessLogic`: the business logic to execute when a DID tries to log in. Will be executed each time the `/auth` endpoint is invoked with a valid signature. If it throws an error, the error message will be returned as part of an HTTP 401 response. _If not present, no business logic will be executed._

`requiredCredentials: string[]`: array of [Verifiable Credential](https://www.w3.org/TR/vc-data-model/) schemas that will be requested as part of the signup process. _If neither `requiredCredentials` and `requiredClaims` are present, no `sdr` will be requested when a user signs up._

`requiredClaims: Claim[]`: array of [Claims](../specs/did-auth#request) that will be requested as part of the signup process. _If neither `requiredCredentials` and `requiredClaims` are present, no `sdr` will be requested when a user signs up._

`signupBusinessLogic: SignupBusinessLogic`: the business logic to execute when a DID tries to sign up. It receives the required [`sdr`](../specs/did-auth#selective-disclosure) as part of the payload. Will be executed each time the `/signup` endpoint is invoked with a valid signature. If it throws an error, the error message will be returned as part of an HTTP 401 response. Should be used to validate the `sdr` against the business needings and/or to save users in any storage for future authentication validation. _If not present, no business logic will be executed._

#### Advanced usage

You can also have full control of your `Express` app and/or modify some specific behaviours, you just need to override or extend our existing factories/classes and use them directly in your express app.

> NOTE: `ExpressDidAuthConfig` extends ALL the config types used in the following classes/factories. Please refer to the [`types.ts`](https://github.com/rsksmart/rif-identity.js/tree/develop/packages/express-did-auth/src/types.ts) file for detailed information about them. 

##### Classes

###### ChallengeVerifier

Is in charge of creating and verifying challenges. This implementation does not save any state, it creates deterministic challenges and recalculates them to check if the received one matches the expected.

```typescript
interface ChallengeVerifier {
  get(did: string): string
  verify(did: string, challenge: string): boolean
}
```

###### RequestCounter

Counts the amount of requests per did. It is designed to allow X amount of requests per Y amount of seconds. Each time Y is elapsed, the counter is reset.

```typescript
interface RequestCounter {
  count(did): void
}
```

###### SessionManager

Keeps the session state. It associates each DIDs with a unique refresh token and some extra session data such as user metadata. 
Is in charge of:
- Create new refresh tokens
- Renew refresh tokens if the received one is still valid (and invalidate prior one)
- Remove refresh tokens from the state once the user log out

```typescript
export interface SessionManager {
  create(did: string): string
  renew(oldToken: string): { refreshToken: string, did: string, metadata: any }
  delete(did: string): void
}
```

##### Factories

###### requestSignupFactory

Receives `challengeVerifier: ChallengeVerifier` and `signupConfig: SignupConfig` as parameters.
Expects the DID in `req.params` as `did`.
If there are `requiredClaims` and/or `requiredCredentials` in the `signupConfig` it creates an `sdr` and signs it.
Creates a challenge associated to the received DID using the `challengeVerifier`.
Responds an HTTP 200 containing `{ challenge, sdr? }`

###### requestAuthFactory

Receives `challengeVerifier: ChallengeVerifier` as parameter.
Expects the DID in `req.params` as `did`.
Creates a challenge associated to the received DID using the `challengeVerifier`.
Responds an HTTP 200 containing `{ challenge }`

###### authenticationFactory

Receives the following parameters:
- `challengeVerifier: ChallengeVerifier`
- `sessionManager: SessionManager`
- `config: AuthFactoryConfig`
- `businessLogic?: AuthenticationBusinessLogic | SignupBusinessLogic`

Expects the `challengeResponse` JWT in `req.body` as `response`.
Verifies the received challenge with the `challengeVerifier`. If valid, executes the `businessLogic` function and then emits an _access token_ (with `config` values) and a _refresh token_ using the `sessionManager`.
If `config.useCookies`, it sets the needed cookies (see [protocol](../specs/did-auth#how-to-send-tokens)).
If not, it responds an HTTP 200 with `{ accessToken, refreshToken }`

> NOTE: this factory is used for both `signup` and `auth` events, because the only difference between them will be the `businessLogic` function to execute.

###### refreshTokenFactory

Receives `sessionManager: SessionManager` and `accessTokenConfig: AuthenticationConfig` as parameters.
If using cookies, will extract the _refresh token_ from them, if not, it expects it as `refreshToken` in `req.body`.
Renews the _refresh token_ with `sessionManager`, and if it is renewed, emits a new _access token_ with the received `accessTokenConfig` values.
If `config.useCookies`, it sets the needed cookies (see [protocol](../specs/did-auth#how-to-send-tokens)).
If not, it returns an HTTP 200 with `{ accessToken, refreshToken }`
If the _refresh token_ has expired, it responds an HTTP 401 with the proper message.

###### logoutFactory

It is a protected endpoint, so it will go through the `expressMiddlewareFactory` before being executed. It assumes that the current user did is present in `req.user.did`.
Receives `sessionManager: SessionManager` as a parameter, which is used to invalidate the _refresh token_ associated to the current `did`.
Responds an HTTP 200.

> NOTE: It does not invalidate the current _access token_, it still be valid until its expiration time.

###### expressMiddlewareFactory

Receives `requestCounter: RequestCounter` and `config: TokenValidationConfig` as parameters.
Depending on cookies config, it extracts the _access token_ from the cookies or from the `Authorization` header.
Verifies the _access token_ JWT (`exp`, `nbf`, `aud`), increments the `requestCounter` and, if everything is ok, continues the request.
If something went wrong, responds with an HTTP 401 and a proper error message.

##### Example

Find below a working example (with no cookies) that uses the basic config, feel free to override/extend/adapt it so it fits your needings:

```typescript
import express from 'express'
import {
  requestSignupFactory, authenticationFactory, requestAuthFactory,
  refreshTokenFactory, expressMiddlewareFactory, logoutFactory
} from '@rsksmart/express-did-auth'
import ChallengeVerifier from '@rsksmart/express-did-auth/lib/challenge-verifier'
import RequestCounter from '@rsksmart/express-did-auth/lib/request-counter'
import SessionManager from '@rsksmart/express-did-auth/lib/session-manager'
import bodyParser from 'body-parser'
import { SimpleSigner } from 'did-jwt'

const privateKey = 'c9000722b8ead4ad9d7ea7ef49f2f3c1d82110238822b7191152fbc4849e1891'

const serviceDid = 'did:ethr:rsk:0x8f4438b78c56B48d9f47c6Ca1be9B69B6fAF9dDa'
const serviceSigner = SimpleSigner(privateKey)
const challengeSecret = 'theSuperSecret'
const serviceUrl = 'https://service.com'

const config: ExpressDidAuthConfig = {
  serviceDid, serviceSigner, challengeSecret, serviceUrl
}

const challengeVerifier = new ChallengeVerifier(config)
const requestCounter = new RequestCounter(config)
const sessionManager = new SessionManager(config)

const app = express()

app.use(bodyParser.json())

app.get('/request-signup', requestSignupFactory(challengeVerifier, config))

app.post('/signup', authenticationFactory(challengeVerifier, sessionManager, config, config.signupBusinessLogic))

app.get('/request-auth', requestAuthFactory(challengeVerifier))

app.post('/auth', authenticationFactory(challengeVerifier, sessionManager, config, config.authenticationBusinessLogic))

app.post('/refresh-token', refreshTokenFactory(sessionManager, config))

app.use(expressMiddlewareFactory(requestCounter, config))

app.post('/logout', logoutFactory(sessionManager))
```

### Run for development

The service source code is hosted in Github, so please refer directly to the [README](https://github.com/rsksmart/rif-identity.js/tree/develop/packages/express-did-auth) and check there the detailed guide to install and test the service locally.