---
title: leanPCS
description: Add a PCS_OPEN precompile, for cheap on-chain verification of STARKs via recursive aggregation
author: TBD
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: Core
created: 2026-11-01
requires: 8141, 8288, MULFIELD EIP
---

## Abstract

This EIP introduces a new precompile `PCS_OPEN` that contracts can call to verify that a committed multilinear polynomial $C$ opens to $Y$ at some point $X$. The underlying polynomial commitment scheme (PCS) is WHIR. This lets users cheaply verify post-quantum STARKs (scalable transparent arguments of knowledge) on chain. `PCS_OPEN` acts as a post-quantum equivalent of the elliptic-curve precompiles used to verify pre-quantum SNARKs. All the heavy WHIR proofs in the block are aggregated, via a STARK, so they don't appear in the block.

## Motivation

Currently, users can verify pre-quantum SNARKs for about 200K gas. Their hash-based post quantum counterparts are much heavier and costly to verify on chain. Thanks to recursive aggregation, the overhead can be amortized to a single STARK per block. One possibility is to expose a new ISA/arithmetization, in which users can register contracts and prove statements. This EIP follows another direction, by realising that the large majority of a STARK verification corresponds to polynomial openings. The idea is then: let Ethereum efficienly prove PCS openings, and keep using the EVM for cheap part of the STARK (PIOP).
<!-- 
## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

| Constant                     | Value  |
|------------------------------|--------|
| `PCS_OPEN_ADDRESS`           | `TBD`  |
| `PCS_SCHEME`                 | `0x12` |
| `MAX_PCS_VARIABLES`          | `32`   |
| `PCS_VERIFICATION_GAS`       | `TBD`  |
| `MAX_PCS_CLAIMS_PER_TX`      | `4`    |
| `MAX_PCS_CLAIMS_PER_WRAPPER` | `4`    |

`DEPS_HASH`, `dependencies(tx)`, `get_deps_hash`, dependency verification frames and mempool wrappers come from EIP-8288. The fields $K = \mathrm{GF}(2^{64})$ and $E = \mathrm{GF}(2^{192})$, and the encoding of their elements as words, come from EIP-XXXX.

### Claims

A claim says: "the multilinear polynomial $f$, committed in `(root, ood_answer)`, takes the value `value` at `point`". $f$ has `n` variables and coefficients in $K$, `point` is in $E^n$, and `value` is in $E$. A claim is encoded as:

```
claim = n (1 byte) || root (32 bytes) || ood_answer (32 bytes) || point (32 * n bytes) || value (32 bytes)
```

The commitment has two parts: `root` is the Merkle root of the WHIR codeword of $f$, and `ood_answer` is the value of $f$ at an out-of-domain point derived from `root`. Field elements are 32-byte big-endian words. A claim is well-formed if `1 <= n <= MAX_PCS_VARIABLES`, if it is `97 + 32 * n` bytes long, and if all its field elements are below $2^{192}$.

Each PCS version defines a verifier `pcs_verify(version, claim, opening_proof)`, whose Fiat-Shamir transcript starts from `claim`. Version `1` is the only version: WHIR at rate 1/4, in the list-decoding regime, with parameters TBD.

### Declaring claims

A transaction declares its claims in an EIP-8288 dependency verification frame. Each claim is one triple `(PCS_SCHEME, DEPS_HASH(claim), version)`, where `version` is a 32-byte big-endian integer. This EIP changes EIP-8288 as follows:

1. A `PCS_SCHEME` triple is valid only if its `version` is supported. It adds `PCS_VERIFICATION_GAS` to the `limits.execution` of its frame.
2. `get_deps_hash` also hashes the scheme of each dependency: it hashes `bytes([t.scheme]) + t.data_hash + t.verification_key_hash` for each dependency `t`.
3. To prove a `PCS_SCHEME` dependency, the recursive STARK takes a witness `(claim, opening_proof)`, and checks that `DEPS_HASH(claim) == data_hash` and that `pcs_verify(version, claim, opening_proof)` holds. In mode 0 mempool wrappers, this pair is the proof of the dependency.
4. For mempool limits, a `PCS_SCHEME` dependency counts as a `LEANSTARK_SCHEME` dependency, with `MAX_PCS_CLAIMS_PER_TX` and `MAX_PCS_CLAIMS_PER_WRAPPER` instead of `MAX_STARKS_PER_TX` and `MAX_LEANSTARK_DEPS_PER_WRAPPER`.

### `PCS_OPEN`

`PCS_OPEN` is a precompile at `PCS_OPEN_ADDRESS`. Its input is `version (1 byte) || claim`. If `version` is not supported, or if `claim` is not well-formed, the call fails. Otherwise, it returns the 32-byte word `1` if the current transaction declared the claim, that is if `(PCS_SCHEME, DEPS_HASH(claim), version)` is in `dependencies(tx)`, and `0` otherwise.

The call costs `30 + 6 * ceil(len(input) / 32)` gas. A valid block proves every claim it declares, so a result of `1` means that the claim is true.

## Rationale

### Why WHIR

Verifying WHIR is mostly checking Merkle paths, which is exactly the part that aggregation removes. WHIR can also be made zero-knowledge by changing only the prover, so private applications can use the same precompile.

### Why claims are hashed

The contract computes `point` and `value` itself while verifying its proof, so the transaction does not need to include them: it only declares a 96-byte triple per claim. `PCS_OPEN` computes `DEPS_HASH` for the contract, because this hash is chosen to be cheap in the aggregation STARK, not in the EVM. The scheme is included in `get_deps_hash` so that dependencies of different schemes cannot collide.

### Upgrades

The field cannot change, but the PCS can. New proofs simply use the new version, but commitments stored in contracts, such as the commitment to a circuit, must be migrated. Following an idea of Justin Drake, a fork adds version $v + 1$, and removes version $v$ after a deprecation period.

To migrate a stored commitment `com` to `com'`, a contract derives a random point $r$ from both commitments by Fiat-Shamir, and requires `PCS_OPEN` to accept both `(v, com, r, y)` and `(v + 1, com', r, y)` for the same $y$. If `com` and `com'` commit to different polynomials, this check passes with probability at most $n / 2^{192}$ (Schwartz–Zippel).

## Backwards Compatibility

This EIP changes `get_deps_hash` of EIP-8288. Nothing else changes for existing transactions, and `LEANSTARK_SCHEME` remains reserved.

## Test Cases

`PCS_OPEN` test vectors are TBD.

## Security Considerations

The security of `PCS_OPEN` relies on EIP-8288 and on the soundness of each PCS version. In the list-decoding regime, `ood_answer` is what binds a commitment to a single polynomial.

The Fiat-Shamir transcript of a WHIR proof starts from the claim only, not from the transcript of the application. This is believed to be sound, but it has not been formally analyzed yet.

`PCS_OPEN` only proves that a claim is true. Each application must still verify the rest of its proof correctly, for example by deriving evaluation points by Fiat-Shamir after its commitments are fixed.

Removing a PCS version makes unmigrated commitments unusable, for example commitments hardcoded in contract code. `MAX_PCS_VARIABLES`, `PCS_VERIFICATION_GAS` and the mempool limits bound denial-of-service risks, as in EIP-8288.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md). -->
