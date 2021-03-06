# Issuer Trust Token prototype

This is a prototype project for verifying a truster Verifiable Credential issuer based on
a token that is included in the credential.

## Background

In decentralized identity, the verifier trusts a credential issued by an issuer because
it trusts the issuer. This trust is often based on a trust framework that is outside the
credential verification flow. For example, the verifier can have a pre-determined list
of issuers that it trusts. The trust list can be maintained either by the verifier itself,
or it can be maintained by a trust framework authority and provided as a service for
verifiers.

When the credential ecosystem includes numerous issuers, as well as frequent changes to the
list of trusted issuers, maintaining the list requires either increasing amount of
maintenance (when maintained by verifiers) or increasing centralization of the system
(if verifiers rely on the central service that maintains the list).

### Goal

This project proposes an alternative approach by introducing a root issuer and including the
issuer trust within each credential. This does not completely remove the need for
maintaining a list as it requires trusted root issuers, but it can significantly reduce
the maintenance effort required.

## Roles:

- **Root Issuer (RI)** is an issuer that is automatically trusted by the Verifier. The Root
Issuer is identified by a DID.

- **Delegate Issuer (DI)** is an issuer that has been authorized by a Root Issuer to issue a
specific Credential Type. The Delegate Issuer is identified by a DID.

- **Issuer Trust Token (ITT)** is a signature generated by the Root Issuer when authorizing the
Delegate Issuer. The signature is calculated from the delegate issuer DID and a Credential Type.

- **Verifier** is an application that receives a Verifiable Credential and wants to verify it.

- **Credential Type** is a machine-readable name of the type of credential received by the Verifier.

## Issuer Trust Token format

The ITT is a JWT generated by the RI and shared with the DI. It can be shared via any channel
as it doesn't include any sensitive information, and it cannot be used by other issuers.

Example VC with an ITT:

```jsonc
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1"
  ],
  "type": ["VerifiableCredential", "ExampleCredential"],
  "issuer": "did:example:sample-delegate-issuer",
  "issuanceDate": "2022-01-04T10:12:00Z",
  "credentialSubject": {
    "id": "did:example:sample-subject",
    "name": "Example",
    "description": "This is an example",
    "itt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ..."
  },
  "proof": {
    // Proof details left out for brevity
  }
}
```

The decoded payload in the `itt` attribute:

```json
{
  "iss": "did:example:sample-root-issuer",
  "sub": "did:example:sample-delegate-issuer",
  "credentialType": "ExampleCredential",
  "iat": "2022-01-01T12:00:00Z",
  "exp": "2022-12-24T12:00:00Z"
}
```

## Verifying Issuer trust

The verifier can verify that the issuer of the VC is trusted with the following steps:

1. Resikve JWT issuer public keys based on the DID in the `iss` claim
2. Validate the JWT signature
3. Check that the `sub` claim matches the DID of the DI
4. Check that the JWT was valid at the time when the VC was issued
5. Check that the `credentialType` claim matches the VC type

## Considerations

### Revocation

ITT has the same limitations as any other use of JWTs. The ITT is a self-contained
token that can be validated without checking with the RI, so revoking the trust from
a DI only takes effect once the current ITT expires.

To mitigate against that, ITT can be a short-lived token (less than 24 hours) and the
DI must request a new ITT whenever the previous one expires and the DI wants to issue
a new credential. This does not however help with revoking trust from credentials that
have been issued previously.
