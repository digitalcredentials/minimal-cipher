# Minimal Cipher _(@digitalcredentials/minimal-cipher)_

Minimal encryption/decryption [JWE](https://tools.ietf.org/html/rfc7516)
library, secure algs only, browser-compatible.

## Table of Contents

- [Security](#security)
- [Background](#background)
- [Install](#install)
- [Usage](#usage)
- [Contribute](#contribute)
- [Commercial Support](#commercial-support)
- [License](#license)

## Security

TBD

## Background

Every version of this library will only offer at most two algorithms
for encryption/decryption: a recommended algorithm and a FIPS-compliant
algorithm. The encryption API will expect the user to specify "recommended"
or "fips" as the version of the algorithm to use, defaulting to "recommended".

In the event that the FIPS-compliant algorithm is the same as the recommended
one in a given version of this library, then that particular version will
use the same algorithm regardless of the user specified "version".

This version of the library will use "XChaCha20-Poly1305" as the "recommended"
version and 256-bit "AES-GCM" as the FIPS-compliant version.

Note: XSalsa20-Poly1305 is an AE (Authenticated Encryption) algorithm, not
an AEAD (Authenticated Encryption and Associated Data) algorithm, making it
incompatible with the current requirements for a
[JWE (JOSE Web Encryption)](https://tools.ietf.org/html/rfc7516)
`protected` clear text header.

This library's API requires an interface for Key Encryption Key (KEKs). This
enables key material that is protected from exfiltration to be used via HSM/SSM
APIs, including Web KMS (TODO: citation needed).

## Install

- Node.js 14+ required.
- [Streams API][] required. Older browsers and Node.js <18 must use a polyfill.
- [Web Crypto API][] required. Older browsers and Node.js 14 must use a polyfill.

To install locally (for development):

```
git clone https://github.com/digitalcredentials/minimal-cipher.git
cd minimal-cipher
npm install
```

## Usage

Pick a Cipher interface (`recommended` or `fips`) and create an instance:

```js
import {Cipher} from '@digitalcredentials/minimal-cipher';

const cipher = new Cipher(); // by default {version: 'recommended'}
```

### Encrypting

To encrypt something (to create a cipher, serialized as a JWE JSON document),
you will need:

* Some data to encrypt (a string, an object, a stream)
* Keys (called Key Agreement Keys, or KAKs for short)

(You'll also need a `keyResolver`, more about that later.)

First, assemble your Key Agreement public keys (you'll be encrypting with them,
and the intended recipient will use the corresponding private keys to decrypt).

Put together a list of `recipients` (essentially, you're listing the `id`s of
public/private key pairs that will be used to encrypt/decrypt the message):

```js
// Retrieve them from config, a ledger, registry or back channel
const keyAgreementKey = await fetchFromSomewhere();

// or derive them from an existing Ed25519 signing key
import {X25519KeyAgreementKey2020} from '@digitalcredentials/x25519-key-agreement-key-2020';
import {Ed25519VerificationKey2020} from '@digitalcredentials/ed25519-verification-key-2020';
const keyPair = await Ed25519VerificationKey2020.generate();

const keyAgreementKey = X25519KeyPair.fromEd25519VerificationKey2020({keyPair});
// If the source key pair didn't have a controller set, don't forget to set one:
keyAgreementKey.controller = did; // The controller's DID
keyAgreementKey.id = `${did}#${keyAgreementKey.fingerprint()}`;

// or derive them from an authentication key extracted from DID Document
const didDoc = await veresDriver.get({did});
const authnKey = didDoc.getVerificationMethod({proofPurpose: 'authentication'});
const edKeyPair = await Ed25519VerificationKey2020.from(authnKey);
const keyPair = X25519KeyPair.fromEd25519VerificationKey2020({keyPair});

const recipient = {
  header: {
    kid: keyAgreementKey.id,
    alg: 'ECDH-ES+A256KW'
  }
}

const recipients = [recipient];
```

You'll also need a `keyResolver`. Notice that `recipients` lists only key IDs,
not the keys themselves. A `keyResolver` is a function that accepts a key ID
and resolves to the public key corresponding to it.

Some example resolvers:

```js
// Basic hardcoded key resolver; you already have the key material
const publicKeyNode = {
  '@context': 'https://w3id.org/security/suites/x25519-2020/v1',
  id: keyAgreementKey.id,
  type: 'X25519KeyAgreementKey2020',
  publicKeyMultibase: keyAgreementKey.publicKeyMultibase
};
const keyResolver = async () => publicKeyNode;
```

```js
// A more advanced resolver based on DID doc authentication keys
const keyResolver = async ({id}) => {
  // Use veres driver to fetch the authn key directly
  const keyPair = await Ed25519VerificationKey2020.from(await veresDriver.get({did: id}));
  // Convert authn key to key agreement key
  return X25519KeyPair.fromEd25519VerificationKey2020({keyPair});
}
```

```js
// Using did-veres-one driver as a resolver for did:v1:nym: DID keys
// TODO: Implement this
```

```js
// Using the did:key method driver as a key resolver
```

Create the JWE:

```js
// To encrypt a string or a Uint8Array
const data = 'plain text';
const jweDoc = await cipher.encrypt({data, recipients, keyResolver});

// To encrypt an object
const obj = {key: 'value'};
const jweDoc = await cipher.encryptObject({obj, recipients, keyResolver});
```

### Decrypting

Decrypt a JWE JSON Document, using a private `keyAgreementKey`:

```js
const data = await cipher.decrypt({jwe, keyAgreementKey});

const object = await cipher.decryptObject({jwe, keyAgreementKey});
```

TODO: Describe the required KEK API:
// `id`, `algorithm`, `wrapKey({unwrappedKey})`, and `unwrapKey({wrappedKey})`

## Contribute

See [the contribute file](https://github.com/digitalbazaar/bedrock/blob/master/CONTRIBUTING.md)!

PRs accepted.

If editing the README, please conform to the
[standard-readme](https://github.com/RichardLitt/standard-readme) specification.

## License

[New BSD License (3-clause)](LICENSE) © Digital Bazaar

[Streams API]: https://developer.mozilla.org/en-US/docs/Web/API/Streams_API
[Web Crypto API]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API
