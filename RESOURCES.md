# Resources

## Standards
- [W3C Web Authentication Level 3](https://www.w3.org/TR/webauthn-3/)
- [FIDO Alliance specifications (CTAP 2.x)](https://fidoalliance.org/specifications/) and [FIDO Metadata Service](https://fidoalliance.org/metadata/)
- [RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md) secp256r1 precompile for rollups
- [EIP-7951](https://eips.ethereum.org/EIPS/eip-7951) secp256r1 precompile for Ethereum L1
- [ERC-4337](https://eips.ethereum.org/EIPS/eip-4337), [ERC-7562](https://eips.ethereum.org/EIPS/eip-7562), [ERC-7579](https://eips.ethereum.org/EIPS/eip-7579), [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271)

## Libraries and implementations
- [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/5.x/api/utils/cryptography) (`P256`, `Base64`, WebAuthn utilities in recent versions)
- [base-org/webauthn-sol](https://github.com/base-org/webauthn-sol), [Coinbase Smart Wallet](https://github.com/coinbase/smart-wallet)
- [Safe modules (passkey)](https://github.com/safe-global/safe-modules)
- [Daimo p256-verifier](https://github.com/daimo-eth/p256-verifier)
- [py_webauthn](https://github.com/duo-labs/py_webauthn)
- [Project Wycheproof](https://github.com/C2SP/wycheproof) test vectors

## Tooling
- [Foundry](https://book.getfoundry.sh) (`vm.signP256` cheatcode)
- Chrome DevTools WebAuthn panel (virtual authenticators)
- [mkcert](https://github.com/FiloSottile/mkcert) for local HTTPS

## Learning
- [passkeys.dev](https://passkeys.dev)
- [FIDO Alliance passkey resources](https://fidoalliance.org/passkeys/)

## Related books in this library
- **account-abstraction-security**, **social-recovery-security**, **web3-supply-chain-security**, **browser-extension-wallet-malware**
