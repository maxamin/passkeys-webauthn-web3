# P-256 Verification On-Chain: RIP-7212 and Solidity Verifiers

Passkeys sign with ECDSA over P-256 (secp256r1). The EVM's built-in `ecrecover` handles only secp256k1. Verifying passkey signatures on-chain therefore needs either a precompile or a Solidity implementation. This chapter covers both, their gas costs, and the correctness traps.

## Options

| Approach | Where | Approximate cost | Notes |
|---|---|---|---|
| RIP-7212 precompile | many L2 rollups that adopted it | about 3,450 gas per the RIP | address `0x100`; behaviour defined by the RIP |
| EIP-7951 precompile | Ethereum L1 from the Fusaka upgrade | higher gas than RIP-7212 per the EIP | designed to be compatible with RIP-7212's interface while fixing edge cases |
| Pure Solidity verifier | any EVM chain | hundreds of thousands of gas | e.g. FreshCryptoLib, Daimo's p256-verifier |
| Library with fallback | any EVM chain | precompile cost if available, else Solidity | e.g. OpenZeppelin `P256` |

Check the exact gas schedule and activation status for each chain you deploy on; they differ and change.

## The precompile interface

RIP-7212 defines `P256VERIFY` at address `0x100`. Input is 160 bytes:

```text
hash (32) | r (32) | s (32) | x (32) | y (32)
```

On success it returns 32 bytes encoding `1`. On failure it returns **empty output**, not `0`. That detail causes bugs: code that does `abi.decode(ret, (uint256))` reverts on failure instead of returning false, and code that only checks `success` of the `staticcall` accepts everything, because a call to a precompile address that does not exist on that chain also "succeeds" with empty output.

```solidity
function verifyPrecompile(bytes32 h, bytes32 r, bytes32 s, bytes32 x, bytes32 y)
    internal view returns (bool)
{
    (bool ok, bytes memory ret) = address(0x100).staticcall(abi.encode(h, r, s, x, y));
    return ok && ret.length == 32 && abi.decode(ret, (uint256)) == 1;
}
```

Never assume the precompile exists. If a contract deployed with the same bytecode across chains hits a chain without it, the call returns empty and must be treated as failure (and ideally fall back to a Solidity verifier).

## Correctness traps

- **Malleability.** For ECDSA, if `(r, s)` is valid then `(r, n - s)` is also valid. If anything uses the signature bytes as a unique identifier (for replay protection or as a mapping key), an attacker can produce a second valid signature. Use nonces, not signature uniqueness, and check whether your library enforces low-s.
- **Range checks.** `r` and `s` must be in `[1, n-1]`. Public keys must be on the curve and not the point at infinity. Precompiles and libraries define their own handling; verify yours.
- **Hash input.** WebAuthn signs `SHA-256(authenticatorData || SHA-256(clientDataJSON))`. The verifier's `hash` argument is that final digest; do not hash again.
- **Public key encoding.** Registration returns a COSE key; extract `x` and `y` correctly (32 bytes each, big-endian, with leading zeros preserved).
- **Gas in ERC-4337 validation.** Solidity fallback verification is expensive; set `verificationGasLimit` accordingly or validation fails.

## Why not just use secp256k1?

WebAuthn allows several algorithms, but platform authenticators in phones and laptops implement P-256 (ES256) because that is what their secure hardware supports and what the FIDO ecosystem standardised on. A wallet cannot ask the user's phone to sign with secp256k1. The chain must therefore meet the authenticator where it is, which is why rollups adopted RIP-7212 and Ethereum L1 followed with EIP-7951. The cost of this choice is a second signature scheme in the account's trust base, with its own libraries, edge cases and gas profile. Treat it as new cryptographic code, not as a drop-in replacement for `ecrecover`.

## Choosing a verifier

- Prefer audited libraries that use the precompile where present and fall back otherwise, such as OpenZeppelin's `P256`.
- Pin the version and read its handling of edge cases.
- Test against standard vectors (Wycheproof includes ECDSA P-256 vectors) to check boundary behaviour.

## Worked example

> **Execution status:** not executed in this book's CI. Commands, code and expected results below are written from documentation, not observed output; verify locally.

Verify a P-256 signature in Foundry with OpenZeppelin's `P256` library. Foundry provides a `vm.signP256` cheatcode for test signatures.

1. Set up:

```bash
forge init p256-lab && cd p256-lab
forge install OpenZeppelin/openzeppelin-contracts
```

2. Generate the public key for a test private key outside Solidity:

```bash
python3 - <<'EOF'
from cryptography.hazmat.primitives.asymmetric import ec
k = ec.derive_private_key(0xA11CE, ec.SECP256R1())
n = k.public_key().public_numbers()
print(hex(n.x), hex(n.y))
EOF
```

3. Write a test using those coordinates:

```solidity
import {Test} from "forge-std/Test.sol";
import {P256} from "@openzeppelin/contracts/utils/cryptography/P256.sol";

contract P256Test is Test {
    bytes32 constant X = 0x0; // paste x from step 2
    bytes32 constant Y = 0x0; // paste y from step 2

    function test_verify() public {
        bytes32 digest = sha256("hello passkey");
        (bytes32 r, bytes32 s) = vm.signP256(0xA11CE, digest);
        assertTrue(P256.verify(digest, r, s, X, Y));
        assertFalse(P256.verify(digest ^ bytes32(uint256(1)), r, s, X, Y));
    }
}
```
4. Malleability check: compute `s2 = n - s` (with `n` the P-256 group order) and test whether your library version accepts `(r, s2)`. Record the result; it determines whether signature bytes can ever be used as identifiers.
5. Gas: run `forge test --gas-report`. In the default local EVM the precompile may be absent and the Solidity fallback is used; run against a node or fork of a chain with RIP-7212 to compare.

## Common pitfalls

- Checking only `success` from the precompile call.
- Decoding empty return data and reverting instead of returning false.
- Using signatures as unique IDs.
- Double-hashing the WebAuthn digest.
- Assuming every chain has the precompile at the same gas cost.
- Too little verification gas for the Solidity fallback in ERC-4337.

## Exercises

1. Write the `verifyPrecompile` function above and test it on a local chain without the precompile; confirm it returns false rather than true.
2. Run Wycheproof ECDSA P-256 SHA-256 vectors through your chosen verifier in a Foundry test and report any disagreements.
3. Measure gas for the Solidity fallback and estimate the extra cost per UserOperation on a chain without the precompile.

## Further reading

- RIP-7212: <https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md>
- EIP-7951: <https://eips.ethereum.org/EIPS/eip-7951>
- OpenZeppelin Contracts `P256`: <https://docs.openzeppelin.com/contracts/5.x/api/utils/cryptography>
- Daimo p256-verifier: <https://github.com/daimo-eth/p256-verifier>
- Project Wycheproof: <https://github.com/C2SP/wycheproof>
