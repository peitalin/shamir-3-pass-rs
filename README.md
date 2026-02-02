# shamir-3-pass

Rust implementation of the Shamir 3-pass (commutative encryption) protocol, designed for native and `wasm32` builds.

Shamir's 3-pass protocol is a *commutative lock* over a symmetric key (KEK) using modular exponentiation.
- Encrypts application data with `ChaCha20Poly1305`; the KEK is turned into an AEAD key via HKDF-SHA256.


## Installation

MSRV: Rust 1.70+

```toml
[dependencies]
shamir-3-pass = "0.5"
```

Or:

```sh
cargo add shamir-3-pass
```

## Quickstart

End-to-end example showing “registration” (store ciphertext + `kek_s`) and “login” (recover KEK and decrypt):

```rust
use shamir_3_pass::{generate_shamir_p_b64u, Shamir3Pass};

// Setup (shared): generate `p` once, persist it, and share it with the other party.
// For real deployments, prefer 2048+ bits. (256-bit is accepted but is not a safe default.)
let p_b64u = generate_shamir_p_b64u(2048).unwrap();
let shamir = Shamir3Pass::new(&p_b64u).unwrap();

// Server: generate long-lived lock keys (e_s, d_s). Persist these securely server-side.
let server = shamir.generate_lock_keys().unwrap();

// Client: encrypt some data under a random KEK.
let (ciphertext, kek) = shamir.encrypt_with_random_kek_key(b"secret").unwrap();

// Client: create a one-time lock (e_c, d_c).
let client = shamir.generate_lock_keys().unwrap();

// Locking step: KEK -> KEK_c -> KEK_cs -> KEK_s (store `ciphertext` + `kek_s` on the server).
let kek_c = shamir.add_lock(&kek, &client.e);        // client adds lock
let kek_cs = shamir.add_lock(&kek_c, &server.e);     // server adds lock
let kek_s = shamir.remove_lock(&kek_cs, &client.d);  // client removes its lock

// Unlock step: KEK_s -> KEK_st -> KEK_t -> KEK (recovered).
let client_login = shamir.generate_lock_keys().unwrap(); // fresh temporary lock
let kek_st = shamir.add_lock(&kek_s, &client_login.e);   // client adds temp lock
let kek_t = shamir.remove_lock(&kek_st, &server.d);      // server removes its lock
let kek_recovered = shamir.remove_lock(&kek_t, &client_login.d); // client removes temp lock

let plaintext = shamir.decrypt_with_key(&ciphertext, &kek_recovered).unwrap();
assert_eq!(plaintext, b"secret");
```

## Protocol overview

Shamir 3-pass uses commutative exponentiation over a shared public modulus `p`:

- Add a lock: `x' = x^e mod p`
- Remove your lock: `x = (x')^d mod p`, where `e*d ≡ 1 (mod p-1)`
- Locks commute: `(x^e_c)^e_s = (x^e_s)^e_c`

This lets a server store ciphertext plus a KEK that remains “server-locked” (`kek_s`), while the client can later recover the KEK without revealing the plaintext to the server.

## Encoding for transport / storage

This crate includes helpers to encode big integers as base64url (unpadded) for storage/transport:

```rust
use shamir_3_pass::{
    decode_biguint_b64u, encode_biguint_b64u, generate_shamir_p_b64u, Shamir3Pass,
};

let p_b64u = generate_shamir_p_b64u(2048).unwrap();
let shamir = Shamir3Pass::new(&p_b64u).unwrap();
let keys = shamir.generate_lock_keys().unwrap();

let e_b64u = encode_biguint_b64u(&keys.e);
let e = decode_biguint_b64u(&e_b64u).unwrap();
assert_eq!(e, keys.e);
```

## WASM

- Randomness uses `getrandom`; the `js` backend is enabled automatically on `wasm32`.
- This crate is Rust-first; it supports `wasm32-unknown-unknown` when used from Rust.

```sh
cargo build --target wasm32-unknown-unknown
```

## Development

```sh
cargo test
```

## Security notes

- This crate has not been professionally audited. Treat it as "use at your own risk", especially in high-assurance environments.
- `Shamir3Pass::new()` validates the *size* of `p`, but does not currently prove `p` is prime; generate `p` with `generate_shamir_p(_b64u)` or validate your own modulus before constructing `Shamir3Pass`.
- Big integer arithmetic (`num-bigint`) is not constant-time; do not assume resistance to timing/side-channel attacks.
- If you expose `add_lock`/`remove_lock` operations to untrusted inputs while using long-lived exponents, consider range/subgroup validation and parameter choices (e.g., a “safe prime” construction) to reduce the risk of small-subgroup style attacks.

If you believe you found a security issue, see `SECURITY.md`.

## License

Licensed under the Apache License, Version 2.0. See `LICENSE`.
