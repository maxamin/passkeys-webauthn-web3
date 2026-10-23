# WebAuthn Signatures in Smart Accounts

A P-256 verifier is only half the job. A passkey signature covers `authenticatorData` and `clientDataJSON`, not the UserOperation hash directly. The smart account must parse these structures and check that they commit to the right operation. This chapter covers what an ERC-4337 or ERC-7579 WebAuthn validator must check and the bugs that appear when it does not.

## What the validator receives

A typical WebAuthn signature blob in a UserOperation contains:

- `authenticatorData` (bytes);
- `clientDataJSON` (string), or a compact form with the prefix and suffix around the challenge;
- indices of the `"challenge"` and `"type"` fields inside `clientDataJSON`;
- `r`, `s`;
- an identifier of which registered key signed (if the account has several).

## Checks, in order

| Check | Why |
|---|---|
| `type` is `"webauthn.get"` | prevents using a registration (`webauthn.create`) signature, or other contexts |
| `challenge` equals base64url(userOpHash) at the claimed index | binds the signature to this operation; the core check |
| UP flag set; UV flag set if policy requires | ensures a human interaction and, with UV, biometric/PIN |
| `rpIdHash` equals expected RP ID hash (optional but recommended) | binds to the wallet's domain |
| origin matches expected value (optional) | defence in depth; costs gas to compare |
| P-256 signature over `sha256(authenticatorData || sha256(clientDataJSON))` | authenticity |
| key is a registered owner of this account | authorisation |

The challenge check is where most bugs hide. The validator must check that the bytes at the given index are exactly `"challenge":"<base64url>"`, that the claimed index actually points inside the JSON at that key and not inside another field's string value, and that the encoding is base64url without padding. Libraries such as Base's `webauthn-sol` (used by Coinbase Smart Wallet) and OpenZeppelin's WebAuthn utilities handle this; review whichever you use against the WebAuthn spec's serialisation rules.

## Reference implementations

- **Coinbase Smart Wallet** supports passkey owners alongside Ethereum addresses, using `webauthn-sol` for verification.
- **Safe passkey module** (`safe-modules` repository) deploys a signer contract per passkey that the Safe treats as an ERC-1271 owner.
- **ERC-7579 validator modules** for WebAuthn exist from several module providers.

Each makes different choices about origin checks, UV requirements, and how keys are stored. When auditing, list these choices explicitly.

## Account design issues

- **Counterfactual addresses.** Accounts are often deployed lazily at an address derived from the passkey's public key. If the derivation omits a chain-specific or factory-specific salt, the same passkey yields the same address on every chain, which is convenient but means signatures must include chain ID (ERC-4337's `userOpHash` does).
- **Multiple owners.** Allowing both an EOA owner and passkey owners is good for recovery but widens the attack surface; owner management functions must require the right signer.
- **Replayable owner changes.** Some designs allow owner changes to be replayed across chains intentionally (to keep owner sets in sync). Review the exact scope of such signatures.
- **Gas.** With a Solidity P-256 fallback, verification may exceed default `verificationGasLimit`; under-estimation causes failures that users see as a broken wallet.
- **ERC-7562 rules.** Validation must obey bundler restrictions; reading external verifier contracts is fine if they are stateless, but storage access rules apply (see **account-abstraction-security**).

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Build a minimal WebAuthn validation function in Foundry and test the challenge binding.

1. In the chapter 3 project, add Base's `webauthn-sol` or OpenZeppelin's WebAuthn library (check which version your OpenZeppelin release ships). The steps below use plain Solidity to show the logic.
2. Build the message exactly as WebAuthn does, in the test:

```solidity
function _clientData(bytes32 opHash) internal pure returns (string memory) {
    return string.concat(
        '{"type":"webauthn.get","challenge":"',
        Base64.encodeURL(abi.encodePacked(opHash)),
        '","origin":"https://wallet.example","crossOrigin":false}'
    );
}

function test_challengeBinding() public {
    bytes32 opHash = keccak256("op-1");
    bytes memory authData = abi.encodePacked(sha256("wallet.example"), uint8(0x05), uint32(0)); // UP|UV
    string memory cd = _clientData(opHash);
    bytes32 digest = sha256(abi.encodePacked(authData, sha256(bytes(cd))));
    (bytes32 r, bytes32 s) = vm.signP256(PK, digest);
    assertTrue(validator.verify(opHash, authData, cd, r, s, X, Y));
    assertFalse(validator.verify(keccak256("op-2"), authData, cd, r, s, X, Y));
}
```

3. Implement `validator.verify` to: check `type`, locate and compare the challenge, check flags, then call `P256.verify`. Use OpenZeppelin's `Base64.encodeURL` for the expected challenge string.
4. Write a negative test where the attacker places `"challenge":"<expected>"` inside a different field, such as the origin string, and the real challenge field holds something else. Your implementation must reject it.
5. Write a negative test with UP cleared (flags `0x04`) and confirm rejection.

## Common pitfalls

- Searching the JSON for the challenge string anywhere instead of at the `challenge` key.
- Accepting standard base64 or padded encodings inconsistently.
- Not checking `type`.
- Not requiring UV where the threat model needs biometric confirmation.
- Using signature bytes for replay protection (malleability, chapter 3).
- Insufficient verification gas with Solidity fallback verifiers.

## Exercises

1. Complete the validator and its tests, including the "challenge inside another field" case.
2. Read the WebAuthn spec's section on `clientDataJSON` serialisation and list the guarantees a verifier can rely on about key order.
3. Review one open-source passkey smart-account implementation and write down its choices for origin, UV, rpIdHash and key storage.

## Further reading

- Base `webauthn-sol`: <https://github.com/base-org/webauthn-sol>
- Coinbase Smart Wallet contracts: <https://github.com/coinbase/smart-wallet>
- Safe modules (passkey): <https://github.com/safe-global/safe-modules>
- W3C WebAuthn Level 3, client data serialisation: <https://www.w3.org/TR/webauthn-3/>
- See **account-abstraction-security**
