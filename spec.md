# DID Method Specification

`did:velocity:v2` is immutable DID method for storing public keys on the Velocity Blockchain. It is similar to did:key and did:jwk in that is supports
only a single key being written to the ledger. The main difference from those existing DID methods is that did:velocity is encoded as a bucket reference
to an on chain contract that ensures that resolution will burn an ethereum NFT 

## DID Format

The Velocity DID format in ABNF form

```
did-velocity-format := "did:velocity:v2:" identifiers
identifiers := identifier / “multi:” 1*(identifier “;”)
identifier: account-id ":" list-id ":" entry-id
account-id = “0x” 1*HEXDIG
list-id: *id-char
entry-id: *id-char
id-char: DIGIT
```

The `identifier` is a location in the velocity credential metadata contract. The contracts state consists of bitlists of fixed size. Each list contains up to 10000 
encrypted keys.

Note that when written to a the Velocity Network DLT the account-id is 0x and 40 hexadecimal characters

When having a DID with `identifiers` that is prefixed with `multi:` then more than one identifier can be used with the end identified by a semi colon “;”. This form is only used during DID resolution.

## DID Operations

### Create

#### To create the DID:

1. Generate the private/public key pair
2. Derive a secret by hashing the credential and running Argon2 algorithm
3. Encrypt the public key using AES-256-GCM
4. Export it as a hex value
5. Add a credential type value
6. Select a position in the metadata list for it be stored
7. Encode the metadataa list as an ethereum url
8. Attach the prefix `did:velocity:v2` that ethereum url

#### To create the DID Document

1. Derive the secret used initially by hashing the credential and running Argon2 algorithm
2. Decrypt the hex value
3. Convert the hex key to a jwk (`json-web-key`)

Generate the json
```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1"
  ],
  "id": "did:velocity:v2:${ethereum-url}",
  "verificationMethod": [
    {
      "id": "did:velocity:v2:${ethereum-url}#0",
      "type": "JsonWebKey2020",
      "controller": "did:velocity:v2:${ethereum-url}",
      "publicKeyJwk": ${json-web-key}
    }
  ],
  "assertionMethod": ["did:velocity:v2:${ethereum-url}#0"],
  "authentication": ["did:velocity:v2:${ethereum-url}#0"],
  "capabilityInvocation": ["did:velocity:v2:${ethereum-url}#0"],
  "capabilityDelegation": ["did:velocity:v2:${ethereum-url}#0"],
  "keyAgreement": ["did:velocity:v2:${ethereum-url}#0"]
}
```

If the JWK contains a `use` property with the value "sig" then the `keyAgreement` property is not included in the DID Document.  If the `use` value is "enc" then _only_ the `keyAgreement` property is included in the DID Document.

The JWK _should_ have the appropriate `use` value set to match the capabilities of the specified `crv`.  For example, the curve `ed25519` is only valid for "sig" use and `X25519` is only valid for "enc" (see [RFC 8037](https://datatracker.ietf.org/doc/html/rfc8037) and the second example below).

The JWK may contain additional custom properties and values which will be accessible only in the `verificationMethod`.  Any additional properties other than `use` (as documented above) are not referenced or used in the generation of the DID Document.

#### To create the DID URL:

Since `did:velocity:v2` only contains a single key, the DID URL fragment identifier is always a fixed `#0` value.

If the JWK contains a `kid` value it is _not_ used as the reference, `#0` is the only valid value.


### Read

1. Remove the prefix `did:velocity:v2:`
2. Decode the remaining string using [Ethereum Transaction URL](https://eips.ethereum.org/EIPS/eip-681)
3. Use the ethereum url is used to lookup the data contained at that location
4. On the transaction include an account cotaining a Velocity Coupon. The coupon will be burnt during resolution.
5. The type and hex value of the encrypted public key will be returned along with the Issuer VC on the metadata list

### Update

Not supported.

### Deactivate

Not supported.

## Examples

### ES256K Key

#### DID
```text
did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432
```

#### KID URL
```text
did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0
```

#### DID Document
```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1"
  ],
  "id": "did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432",
  "verificationMethod": [
    {
      "id": "did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0",
      "type": "JsonWebKey2020",
      "controller": "did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432",
      "publicKeyJwk": {
        "crv": "ES256K",
        "kty": "EC",
        "x": "acbIQiuMs3i8_uszEjJ2tpTtRM4EU3yz91PH6CdH2V0",
        "y": "_KcyLj9vWMptnmKtm46GqDz8wf74I5LKgrl2GzH3nSE"
      }
    }
  ],
  "assertionMethod": ["did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0"],
  "authentication": ["did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0"],
  "capabilityInvocation": ["did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0"],
  "capabilityDelegation": ["did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0"],
  "keyAgreement": ["did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432#0"]
}
```

## Security and Privacy Considerations

Since this velocity method is very similar to the DID Key method, see also [did-key](https://w3c-ccg.github.io/did-method-key/#security-and-privacy-considerations)

### Security

Since the contract on chain is immutable is no support for key rotation, if the key is compromised then the identifier becomes unusable and unrecoverable. Therefore this method should only be used when the private key will not be used again. 

There is no provided means of cryptographically verifying possession of the public key material, any such verification must be performed separately by applications using a sufficient challenge-response protocol.

### Privacy

Using the same DID Velocity identifier with multiple different entities will enable those entities to correlate the usage to the same subject.
