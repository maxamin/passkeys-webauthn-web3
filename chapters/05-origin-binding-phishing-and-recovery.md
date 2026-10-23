# Origin Binding, Phishing Resistance and Device Loss

Passkeys are called phishing-resistant because browsers enforce origin binding: a credential registered for `wallet.example` cannot be used by `wallet-example.app`. That stops a whole class of phishing. It does not stop an attacker who controls the legitimate origin, and it creates new questions about multiple front-ends and lost devices. This chapter covers both.

## How origin binding works

- At registration the credential is bound to an **RP ID**, a registrable domain such as `wallet.example`.
- At authentication the browser only allows pages whose origin is that domain or a subdomain to request the credential.
- The browser writes the actual **origin** into `clientDataJSON`, which the authenticator signs.

A phishing site on a different domain therefore cannot obtain an assertion, and cannot forge `clientDataJSON` without invalidating the signature.

## What it does not stop

| Attack | Why origin binding does not help |
|---|---|
| Compromised wallet front-end (malicious JS, CDN or dependency compromise) | the malicious code runs on the real origin |
| Malicious browser extension with page access | runs in the page context on the real origin |
| DNS or domain hijack | the attacker serves the real domain |
| Approval phishing inside the real wallet (signing a malicious transaction the user was lured into) | the wallet asks for a signature on whatever it was told |
| Malware on the device | may drive the browser |

The passkey prompt shows the site, not the transaction. Unlike a hardware wallet, there is no independent display. The wallet front-end is therefore part of the signing trust base. Protect it with subresource integrity, strict content security policy, reproducible builds, and change monitoring (see **web3-supply-chain-security** and **browser-extension-wallet-malware**).

## RP ID choices for web3

Who is the relying party? Options:

1. **The wallet's domain.** Users sign at the wallet's site or in its app. Dapps connect via the wallet (popup, iframe or SDK). This is the common design for passkey smart wallets.
2. **Each dapp's domain.** Each dapp would have its own credential for the same account, which complicates account ownership.
3. **Related origins.** WebAuthn Level 3 defines related origin requests: an RP can list other origins allowed to use its RP ID in a `/.well-known/webauthn` file. Useful for a wallet with several domains; every listed origin joins the trust base.
4. **Native apps.** Mobile apps use platform associations (for example Apple associated domains, Android Digital Asset Links) to claim an RP ID. Misconfigured associations can let the wrong app use credentials.

If the wallet's domain is lost (expiry, registrar compromise), the credentials still exist but the attacker can request signatures from users who visit. Domain security (registrar locks, DNSSEC where possible, monitoring) is wallet security.

## Device loss and recovery

A passkey-only account recovers in one of these ways:

| Situation | Path |
|---|---|
| Synced passkey, lost one device | other devices or platform account restore it |
| Synced passkey, lost platform account | gone unless another signer exists |
| Device-bound passkey lost | gone unless another signer exists |
| Domain lost | credentials unusable at the new attacker-controlled site; account needs signers usable elsewhere |

Therefore every passkey wallet with meaningful value needs at least one additional signer: a second passkey from a different provider, a hardware wallet EOA owner, or a guardian set with delay. Recovery design is covered in **social-recovery-security**.

## Domain independence of the account

Because the on-chain account accepts any valid signature from a registered key, the account itself does not care about domains, except where the validator checks `rpIdHash` or origin. If it does check them, losing the domain could lock the account. If it does not, a passkey registered elsewhere with the same key could sign (not possible in practice since keys are per-RP, but the contract should be explicit). Decide and document which you want.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Show that origin binding blocks a lookalike domain, and that the contract-side origin check behaves as designed. Run locally only.

1. Add two names to `/etc/hosts`: `127.0.0.1 wallet.test` and `127.0.0.1 wa11et.test`. Serve the chapter 1 lab page over HTTPS for both, using a local CA such as `mkcert`.
2. Register a credential on `https://wallet.test` with `rp.id: "wallet.test"`.
3. On `https://wa11et.test`, call `navigator.credentials.get` with `rpId: "wallet.test"`. The browser refuses because the page's origin is not within that RP ID.
4. On `https://wallet.test`, obtain an assertion and paste `authenticatorData`, `clientDataJSON` and the signature into the Foundry validator from chapter 4, configured to require `rpIdHash == sha256("wallet.test")`. It should verify.
5. Change the validator's expected RP ID to `wa11et.test` and confirm verification fails, demonstrating the contract-side check.
6. Write down what happens to the account if `wallet.test` is lost, under each validator configuration.

## Common pitfalls

- Treating passkeys as protection against front-end compromise.
- Listing many related origins without considering each as a signing surface.
- Passkey-only accounts with no second signer.
- Hard-coding origin checks without a migration path if the domain changes.
- Weak registrar and DNS security for the wallet domain.

## Exercises

1. Draw the trust base for a passkey wallet: every party or component that, if compromised, can obtain a signature.
2. Write a `/.well-known/webauthn` file for a wallet with two domains and list the new risks introduced.
3. Add a hardware-wallet EOA as a second owner to your Foundry account and test that it can rotate the passkey owner after a delay.

## Further reading

- W3C WebAuthn Level 3, RP ID and related origins: <https://www.w3.org/TR/webauthn-3/>
- passkeys.dev, related origin requests guide: <https://passkeys.dev>
- mkcert: <https://github.com/FiloSottile/mkcert>
- MDN, Subresource Integrity and Content Security Policy
- See **social-recovery-security**, **web3-supply-chain-security** and **browser-extension-wallet-malware**
