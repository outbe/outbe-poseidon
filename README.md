# Outbe Poseidon

[![crates.io](https://img.shields.io/crates/v/outbe-poseidon.svg)](https://crates.io/crates/outbe-poseidon)
[![release](https://img.shields.io/github/v/release/outbe/outbe-poseidon.svg)](https://github.com/outbe/outbe-poseidon/releases)
[![CI](https://github.com/outbe/outbe-poseidon/actions/workflows/check.yml/badge.svg)](https://github.com/outbe/outbe-poseidon/actions/workflows/check.yml)

<!-- cargo-rdme start -->

**outbe-poseidon** implements the [Poseidon](https://eprint.iacr.org/2019/458)
hash in Rust for [Outbe](https://github.com/outbe).
It uses the [light-poseidon](https://github.com/Lightprotocol/light-poseidon/)
library as its basis.

## Parameters

The library provides pre-generated parameters for the BN254 curve.
It also supports other parameters if developers generate the round constants.

The library provides these parameters:

* *x^5* S-boxes
* Width: *2 ≤ t ≤ 13*
* Number of inputs: *1 ≤ n ≤ 12*
* Number of full rounds: 8
* Number of partial rounds for each *t*: *[56, 57, 56, 60, 60, 63, 64, 63, 60, 66, 60, 65]*

## Output type

`Poseidon` implements the `PoseidonHasher` trait.
Its `hash` method returns a field element of a type that implements
`ark_ff::PrimeField`.
This result might be useful for immediate processing with another library
that supports `ark_ff::PrimeField` types.

## Examples

This example uses `PoseidonHasher` to return an `ark_ff::PrimeField` element:

```rust
use ark_bn254::Fr;
use ark_ff::PrimeField;
use outbe_poseidon::{Poseidon, PoseidonHasher};

let mut poseidon = Poseidon::<Fr>::new_circom(2).unwrap();

let input1 = Fr::from_le_bytes_mod_order(&[1u8; 32]);
let input2 = Fr::from_le_bytes_mod_order(&[2u8; 32]);

let hash = poseidon.hash(&[input1, input2]).unwrap();

// Use `hash` in your application.
```

## Poseidon2

`Poseidon2` implements a separate, generic **Poseidon2** hash with BN254 support.
Its output matches Noir's in-circuit `poseidon2` bit for bit.
It uses Barretenberg's permutation and sponge.
Use it outside a circuit when the hash must match a Poseidon2 result from a circuit.

Poseidon2 uses a different construction from the Circom-compatible `Poseidon`.
The two implementations share no parameters.
`Poseidon2::<Fr>::new()` provides BN254 support.
Other fields require their own constants.

```rust
use ark_bn254::Fr;
use outbe_poseidon::{Poseidon2, PoseidonHasher};

let mut poseidon2 = Poseidon2::<Fr>::new();
let _hash = poseidon2.hash(&[Fr::from(1u64), Fr::from(2u64)]).unwrap();
```

## Field Arithmetic

This library uses [ark-ff](https://github.com/arkworks-rs/algebra) for field arithmetic.
The ark-ff library includes an academic disclaimer.
Many projects use ark-ff in production, including:

* [Aleo](https://github.com/AleoNet/snarkVM)
* [Penumbra](https://github.com/penumbra-zone/penumbra)
* [Mina (Kimchi)](https://github.com/o1-labs/proof-systems)
* [Espresso Systems](https://github.com/EspressoSystems)

## Implementation

The implementation is compatible with the
[original SageMath implementation](https://extgit.iaik.tugraz.at/krypto/hadeshash/-/tree/master/).
These implementations also influenced its design:

* [circomlibjs](https://github.com/iden3/circomlibjs)
* [zero-knowledge-gadgets](https://github.com/webb-tools/zero-knowledge-gadgets)
* [light-poseidon](https://github.com/Lightprotocol/light-poseidon/)

## Security

[Veridise](https://veridise.com/) audited this library.
You can read the [audit report](https://github.com/Lightprotocol/light-poseidon/blob/main/assets/audit.pdf).

<!-- cargo-rdme end -->

## Verifying releases

Releases tagged from `v0.3.5` onward ship sigstore cosign signatures + SLSA build-provenance attestations for every artifact. See [SECURITY.md](SECURITY.md) for the threat model and the copy-pasteable verify recipe.

Quick check:

```sh
TAG=v0.3.5
ARTIFACT=outbe-poseidon-${TAG#v}.crate
gh release download "$TAG" --repo outbe/outbe-poseidon \
  --pattern "$ARTIFACT" --pattern "$ARTIFACT.sig" --pattern "$ARTIFACT.pem"
cosign verify-blob \
  --certificate "$ARTIFACT.pem" --signature "$ARTIFACT.sig" \
  --certificate-identity-regexp \
    '^https://github\.com/outbe/outbe-poseidon/\.github/workflows/release\.yml@refs/(heads/main|tags/v[0-9]+\.[0-9]+\.[0-9]+)$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  "$ARTIFACT"
```
