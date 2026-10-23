# Synced versus Device-Bound Passkeys

Not all passkeys live in the same place. Some never leave a security key's chip. Others are copied, end-to-end encrypted, to every device in a user's Apple, Google or password-manager account. For login, this is mostly a usability trade-off. For a wallet that holds money, it decides who can ultimately sign. This chapter compares the two and shows how to tell them apart.

## Two families

| Property | Device-bound | Synced |
|---|---|---|
| Where the key lives | one authenticator (security key, TPM, some platform modes) | a sync fabric (iCloud Keychain, Google Password Manager, third-party managers) and every enrolled device |
| Loss of device | credential lost | credential survives on other devices or restored from account |
| Compromise path | physical theft plus PIN/biometric, or authenticator bug | takeover of the sync account and its recovery, malware on any enrolled device, provider issues |
| Attestation | often available and meaningful | usually absent or not tied to a specific hardware model |
| Flags | BE = 0 | BE = 1, BS = 1 when currently backed up |
| Exportability | no | portable between providers under newer credential exchange work |

Backup eligibility (BE) and backup state (BS) are reported in `authenticatorData` flags. A wallet can read them at registration to learn whether it has a synced credential.

## Security of the sync fabric

Platform vendors describe end-to-end encryption for synced passkeys, protected by device screen locks or account-recovery keys. The practical security questions are about account access, not cryptography:

- Can an attacker who takes over the Apple ID or Google account (for example with a SIM swap plus social engineering of support) add a new device and obtain the passkey?
- What does the platform require to add a new device: another trusted device, the old device's passcode, a recovery key?
- Is the user's account protected with hardware security keys or only with SMS?

Each vendor documents its own process; read the current documentation rather than assuming. For high-value wallets, assume that synced passkey security equals the weakest recovery path of the sync account. See **social-recovery-security** for recovery attacks generally.

## Cross-device use

WebAuthn hybrid transport (often shown as "use a phone" with a QR code) lets a phone act as an authenticator for a laptop. The protocol uses Bluetooth proximity to make remote relaying harder. For wallets this means a user can sign a transaction on a desktop with a passkey on a phone; the phone's prompt still shows only the site name.

## Design options for wallets

1. **Synced passkey as the only signer.** Easiest onboarding; security equals the platform account. Suitable for small balances.
2. **Synced passkey plus a recovery or co-signer.** For example, a second passkey on a security key, or a guardian set (chapter 5).
3. **Device-bound passkey for high-value actions.** Use a security key, with attestation, for transfers above a limit, and a synced passkey for small ones. Smart accounts can encode this policy.
4. **Multiple passkeys per account.** Register a passkey per device or provider so that losing one is not fatal.

Spending policies like option 3 require the smart account to know which key signed and what its limits are; this is natural in modular account designs (see **account-abstraction-security**).

## Privacy note

Credential public keys and IDs are per-RP, so passkeys do not link a user across sites. But a smart account's on-chain public key is public forever and the same across chains if the same passkey is used. Consider that a stable on-chain identifier.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Detect synced versus device-bound credentials by parsing `authenticatorData` flags at registration.

1. Capture a registration response from the chapter 1 lab page (`cred.response.getAuthenticatorData()` in Chrome returns the authenticator data as an ArrayBuffer). Hex-encode it and save it.
2. Parse the flags byte with Python:

```python
import sys
ad = bytes.fromhex(sys.argv[1])
flags = ad[32]
names = {0:"UP", 2:"UV", 3:"BE", 4:"BS", 6:"AT", 7:"ED"}
print({n: bool(flags >> b & 1) for b, n in names.items()})
print("signCount", int.from_bytes(ad[33:37], "big"))
```

3. Compare results from: the DevTools virtual authenticator, a platform passkey on a phone or laptop with sync enabled, and a hardware security key. Typically synced passkeys report BE and BS set, and security keys report BE clear.
4. Write a registration policy: "accept BE=1 credentials for limits up to X; require BE=0 with acceptable attestation for higher limits". Implement it in your server or wallet code, and test it against your captured samples.

## Common pitfalls

- Treating synced passkeys as equivalent to hardware wallets.
- Ignoring BE/BS flags at registration.
- Relying on `signCount` to detect cloning; synced passkeys commonly report 0.
- A single passkey with no second signer or recovery.
- Assuming platform account recovery is strong without checking the user's actual settings.

## Exercises

1. Register the same account with three authenticator types and tabulate flags, AAGUID and attestation format.
2. Read one platform vendor's documentation on adding a device to a passkey sync account and list the factors an attacker would need.
3. Design a smart-account spending policy that uses a synced passkey under a daily limit and a device-bound key above it; write its rules in pseudocode.

## Further reading

- W3C WebAuthn Level 3, authenticator data flags: <https://www.w3.org/TR/webauthn-3/>
- FIDO Alliance passkey resources: <https://fidoalliance.org/passkeys/>
- passkeys.dev: <https://passkeys.dev>
- See **social-recovery-security** and **account-abstraction-security**
