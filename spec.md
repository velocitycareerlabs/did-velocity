# DID Method Specification

## Introduction 

The world does not need another DID method, and yet the Velocity Network Foundation decided it needed one. What are the reasons for this? This first is that the combination of requirements that Velocity Network needed for the inventing a DID method for the credentials that were issued on the network

1.	For VC verification keys
2.	Are composable
3.	Decentralized hosting
4.	Immutable
5.	Free creation
6.	Optional fees when read

`did:velocity:v2` is immutable DID method for storing public keys on the Velocity Blockchain. One way it is similar to `did:key` and `did:jwk` in that is supports
only a single key at a time, though in this case they are written to a ledger.

## How it works

When creating a Velocity DID the method stores only a single encrypted key and a credential type in binary format on-chain. They need to be created when the credential is accepted by the holder. Creating the data on-chain requires signing a Velocity Distributed Ledger transaction using the Issuer’s network registered distributed ledger key. Modifications are not permitted, meaning that each VC verification key is static and immutable.

The resolution of the DID Document can be for one or more verifiable credentials and, therefore, can contain multiple keys. Furthermore, the resolution can be restricted to transactions that include payment through an on-chain payment-receipt NFT. During resolution, the payment-receipt NFT will be “burnt,” ensuring single use. The Velocity DID method therefore enables payment gating of verification processes. 


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
2. Derive a secret by running [Encryption Key Derivation](#encrpytion-key-derivation)
3. Encrypt the public key using AES-256-GCM
4. Export it as a hex value
5. Add a credential type value
6. Select a position in the metadata list for it be stored
7. Encode the metadataa list as an ethereum url
8. Attach the prefix `did:velocity:v2` that ethereum url

#### Encryption Key Derivation
1.	Pick the properties `credentialSubject`, `validFrom`, `expirationDate` and `validUntil`into a new property
2.	Run [RFC8785](https://tools.ietf.org/html/rfc8785) JSON canonicalization.
3.	Run the Sha256 hash algorithm on the result
4.	Run the Argon2 algorithm on the reuslt

#### To create the DID URL:

Since `did:velocity:v2` only contains a single key, the DID URL fragment identifier is always a fixed `#0` value.

If the JWK contains a `kid` value it is _not_ used as the reference, `#0` is the only valid value.

### Read

#### Single Key

1. Remove the prefix `did:velocity:v2:`
2. Decode the remaining string using [Ethereum Transaction URL](https://eips.ethereum.org/EIPS/eip-681)
3. Use the ethereum url is used to lookup the data contained at that location
4. (optional) Include the id of a [NFT - ERC721](https://eips.ethereum.org/EIPS/eip-721) compatible token. The token will be burnt during resolution.
5. The type and hex value of the encrypted public key will be returned along with the Issuer VC on the metadata list

#### Multiple Keys

1. Remove the prefix `did:velocity:v2:multi`
2. Extract each separate key identifier by splutting on ";"
3. Run for each identifier steps 2-5 of the [Single Key algorithm](#single-key)

#### To create the DID Document

2. Derive the secret used initially by running [Encryption Key Derivation](#encrpytion-key-derivation)
3. Decrypt the hex value of the encrypted public key using AES-256-GCM
4. Convert the hex key to a jwk (`json-web-key`)

Generate the json
```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1"
  ],
  "id": "did:velocity:v2:${identifier}",
  "verificationMethod": [
    {
      "id": "did:velocity:v2:${ethereum-url}#0",
      "type": "JsonWebKey2020",
      "controller": "did:velocity:v2:${identifier}",
      "publicKeyJwk": ${json-web-key}
    }
  ],
  "assertionMethod": ["did:velocity:v2:${identifier}#0"],
  "authentication": ["did:velocity:v2:${identifier}#0"],
  "capabilityInvocation": ["did:velocity:v2:${identifier}#0"],
  "capabilityDelegation": ["did:velocity:v2:${identifier}#0"],
  "keyAgreement": ["did:velocity:v2:${identifier}#0"]
}
```

The JWK may contain additional custom properties and values which will be accessible only in the `verificationMethod`.  Any additional properties other than `use` (as documented above) are not referenced or used in the generation of the DID Document.

### Update

Not supported.

### Deactivate

Not supported.

## Examples

### ES256K Key

#### DID for single key
```text
did:velocity:v2:0x235f24005b553e50c873ef847cf51a298e5d23ff:3:5432
```

#### DID for multi key
```text
did:velocity:v2:multi:0x056aa5dc5b1cf09529c39f56348ddc46f340ec7d:253869613654622:3262;0xef988b59498b9663909b508123675733cad647cc:51243483654409:1549;
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

This DID velocity method is immutable just like DID:KEY or DID:JWK.

See also [did-key](https://w3c-ccg.github.io/did-method-key/#security-and-privacy-considerations) & [did-jwk](https://github.com/quartzjer/did-jwk/blob/main/spec.md#security-and-privacy-considerations)

### Security

Since the `did:velocity` on chain is immutable is no support for key rotation, if the key is compromised then the identifier becomes unusable and unrecoverable. Therefore this method should only be used when the private key will not be used again. 

There is no provided means of cryptographically verifying possession of the public key material, any such verification must be performed separately by applications using a sufficient challenge-response protocol.

### Privacy

Using the same DID Velocity identifier with multiple different entities will enable those entities to correlate the usage to the same subject.

