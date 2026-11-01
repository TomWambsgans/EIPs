---
title: Binary Field Multiplication Opcode
description: Add a MULFIELD instruction, multiplication in GF(2^192), for efficient verification of binary-field STARKs
author: TBD
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: Core
created: 2026-11-01
---

## Abstract

This EIP adds `MULFIELD`, an instruction that multiplies two elements of the binary field $\mathrm{GF}(2^{192})$. Addition in this field is `XOR` (`0x18`).

## Motivation

Verifying a binary-field STARK (post-quantum scalable transparent argument of knowledge) requires multiplications in a binary field. The Ethereum Virtual Machine (EVM) can add in such a field, with `XOR`, but emulating one multiplication with existing instructions costs more than 1,000 gas.

## Specification

Let $K = \mathrm{GF}(2)[Y] / (Y^{64} + Y^4 + Y^3 + Y + 1)$ and $E = K[X] / (X^3 + X + 1)$. An element of $K$ is a 64-bit integer whose bit $j$ is the coefficient of $Y^j$. An element $a + bX + cX^2$ of $E$ is the word `a | (b << 64) | (c << 128)`.

`MULFIELD` (`0x1f`) pops `a` and `b`, ignores their top 64 bits, and pushes their product in $E$. It costs 10 gas.

## Rationale

From the point of view of a PCS (Polynomial Commitment Scheme), like WHIR, 192 bits is large enough to get provable 128 bits of security in the List Decoding regime. TBD what about a 128 bits field ?

## Backwards Compatibility

The opcode was previously undefined, and executing it halted exceptionally.

## Test Cases

| `a` | `b` | `MULFIELD(a, b)` |
|-----|-----|------------------|
| `0x8000000000000000` | `0x2` | `0x1b` |
| `0x10000000000000000` | `0x100000000000000000000000000000000` | `0x10000000000000001` |
| `0x100000000000000000000000000000000` | `0x100000000000000000000000000000000` | `0x100000000000000010000000000000000` |
| `0x0123456789abcdeffedcba98765432100f1e2d3c4b5a6978` | `0x8796a5b4c3d2e1f00123456789abcdef1032547698badcfe` | `0x7da2e3b52de3b3f4793532f0de83952e8394b1a39681a4de` |
| `0xffffffffffffffff000000000000000000000000000000000000000000000002` | `0x8000000000000000` | `0x1b` |

## Reference Implementation

```python
def mul_k(a, b):  # multiplication in K
    r = 0
    for i in range(64):
        if (b >> i) & 1:
            r ^= a << i
    for i in range(126, 63, -1):  # reduce by Y^64 + Y^4 + Y^3 + Y + 1
        if (r >> i) & 1:
            r ^= ((1 << 64) | 0b11011) << (i - 64)
    return r

def mulfield(a, b):  # multiplication in E
    x = [(a >> (64 * i)) & (2**64 - 1) for i in range(3)]
    y = [(b >> (64 * i)) & (2**64 - 1) for i in range(3)]
    c = [0] * 5
    for i in range(3):
        for j in range(3):
            c[i + j] ^= mul_k(x[i], y[j])
    for d in (4, 3):  # X^d = X^(d-2) + X^(d-3)
        c[d - 2] ^= c[d]
        c[d - 3] ^= c[d]
    return c[0] | (c[1] << 64) | (c[2] << 128)
```

## Security Considerations

TBD

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
