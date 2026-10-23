# Glossary

- **WebAuthn:** W3C API for creating and using public-key credentials in browsers.
- **Passkey:** discoverable WebAuthn credential, often synced across a user's devices.
- **Relying Party (RP) / RP ID:** the service that registers credentials and the domain they are bound to.
- **Authenticator:** device or software that holds credential private keys and signs assertions.
- **clientDataJSON:** browser-built JSON containing type, challenge and origin, hashed into the signed message.
- **authenticatorData:** authenticator-built bytes with rpIdHash, flags and sign counter.
- **UP / UV flags:** user presence and user verification indicators.
- **BE / BS flags:** backup eligibility and backup state; indicate a synced credential.
- **Attestation / AAGUID:** proof of authenticator model and the model identifier.
- **P-256 (secp256r1):** NIST elliptic curve used by ES256 passkey signatures.
- **RIP-7212 / EIP-7951:** precompiles for P-256 signature verification on rollups and Ethereum L1.
- **Signature malleability:** existence of a second valid ECDSA signature `(r, n - s)` for the same message.
- **Related origins:** WebAuthn mechanism allowing listed origins to use an RP ID.
