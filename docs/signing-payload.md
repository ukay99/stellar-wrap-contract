# Canonical Signed Payload Encoding

This document defines the exact byte layout that the backend must sign and that
the `mint_wrap` entry-point verifies with `ed25519_verify`.

> **⚠ WARNING — field order is load-bearing.**
> The contract concatenates each XDR-encoded field in the fixed order shown below
> and passes the resulting byte string directly to `ed25519_verify`.
> Changing the order, omitting a field, or encoding any field differently will
> produce a different message digest and the signature check **will always fail**,
> causing every `mint_wrap` call to be rejected with
> `Error(Contract, #5)` (`InvalidSignature`).

---

## Algorithm

```
payload = XDR(contract_address)
        ‖ XDR(user_address)
        ‖ XDR(period)
        ‖ XDR(archetype)
        ‖ XDR(data_hash)

signature = Ed25519Sign(admin_private_key, payload)
```

`‖` denotes byte-level concatenation. There is no length prefix, separator, or
framing between fields.

---

## Field order and XDR encoding

| # | Field | Rust type | XDR encoding rule |
|---|-------|-----------|-------------------|
| 1 | `contract_address` | `Address` (contract) | `soroban_sdk::xdr::ToXdr` on the `Env`-resolved contract address |
| 2 | `user_address` | `Address` (account) | `soroban_sdk::xdr::ToXdr` on the caller address |
| 3 | `period` | `u64` | `soroban_sdk::xdr::ToXdr` — big-endian 8-byte unsigned integer wrapped in XDR `Uint64` |
| 4 | `archetype` | `Symbol` | `soroban_sdk::xdr::ToXdr` — XDR `ScSymbol` (4-byte length prefix followed by UTF-8 bytes, padded to a 4-byte boundary) |
| 5 | `data_hash` | `BytesN<32>` | `soroban_sdk::xdr::ToXdr` — XDR opaque fixed-length 32 bytes |

### Period encoding

`period` is an integer in `YYYYMM` format (e.g. `202401` for January 2024).
It is encoded as a plain XDR `Uint64` — the semantic meaning is irrelevant to
the encoding. Valid range: `202401`–`210012`.

### Archetype encoding

`archetype` is a short Soroban `Symbol` (up to 32 characters). It is serialised
with `ToXdr` which produces an `ScVal` of type `ScValType::Symbol`. The byte
layout is:

```
XDR discriminant (4 bytes, big-endian u32, value 14 for SCV_SYMBOL)
XDR string length (4 bytes, big-endian u32)
UTF-8 bytes of the symbol
padding to next 4-byte boundary (0–3 zero bytes)
```

---

## Reference implementation

The contract builds the payload in `src/mint.rs`:

```rust
fn build_payload(
    e: &Env,
    contract: &Address,
    user: &Address,
    period: u64,
    archetype: &Symbol,
    data_hash: &BytesN<32>,
) -> Bytes {
    let mut payload = Bytes::new(e);
    payload.append(&contract.to_xdr(e));         // field 1
    payload.append(&user.clone().to_xdr(e));     // field 2
    payload.append(&period.to_xdr(e));           // field 3
    payload.append(&archetype.clone().to_xdr(e)); // field 4
    payload.append(&data_hash.clone().to_xdr(e)); // field 5
    payload
}
```

The test helper `sign_payload` in `src/test.rs` mirrors this construction
exactly and can be used as a reference when implementing the signing service.

---

## Test vectors

The test suite in `src/test.rs` provides several exercisable vectors.
The table below documents the inputs used by `test_minting_flow` (signing key
seed `[1u8; 32]`) and `test_mint_emits_event` (signing key seed `[2u8; 32]`),
which are both deterministic within the Soroban test environment.

### Vector 1 — `test_minting_flow`

| Field | Value |
|-------|-------|
| Signing key seed | `[0x01; 32]` (all bytes = 1) |
| `period` | `202401` |
| `archetype` | `"arch"` (`symbol_short!("arch")`) |
| `data_hash` | `[0x2A; 32]` (all bytes = 42) |
| Expected result | `mint_wrap` succeeds; `get_wrap` returns a record with `data_hash == [0x2A; 32]` |

### Vector 2 — `test_mint_emits_event`

| Field | Value |
|-------|-------|
| Signing key seed | `[0x02; 32]` (all bytes = 2) |
| `period` | `202401` |
| `archetype` | `"arch"` |
| `data_hash` | `[0x01; 32]` (all bytes = 1) |
| Expected result | `mint_wrap` succeeds; emitted event has topics `["mint", user, 202401]` and data `"arch"` |

### Vector 3 — wrong signature is rejected

Any byte modification to the payload (including reordering fields) produces a
different message. The contract will panic with `Error(Contract, #5)`.
`test_duplicate_period_fails` and `test_invalid_period_zero_fails` indirectly
demonstrate this: a stale or malformed signature always causes an early abort.

### Reproducing vectors in Rust

```rust
use ed25519_dalek::{Signer, SigningKey};
use soroban_sdk::{symbol_short, xdr::ToXdr, Address, Bytes, BytesN, Env};

fn sign_payload(
    env: &Env,
    signer: &SigningKey,
    contract: &Address,
    user: &Address,
    period: u64,
    archetype: &soroban_sdk::Symbol,
    data_hash: &BytesN<32>,
) -> BytesN<64> {
    let mut payload = Bytes::new(env);
    payload.append(&contract.to_xdr(env));
    payload.append(&user.clone().to_xdr(env));
    payload.append(&period.to_xdr(env));
    payload.append(&archetype.clone().to_xdr(env));
    payload.append(&data_hash.clone().to_xdr(env));

    let mut buf = [0u8; 512];
    let len = payload.len() as usize;
    payload.copy_into_slice(&mut buf[..len]);

    let sig = signer.sign(&buf[..len]);
    BytesN::from_array(env, &sig.to_bytes())
}
```

---

## Error reference

| Code | Name | Triggered when |
|------|------|----------------|
| `#3` | `Unauthorized` | `user.require_auth()` fails |
| `#5` | `InvalidSignature` | `ed25519_verify` rejects the signature (wrong payload order, wrong key, corrupted bytes) |
| `#6` | `InvalidPeriod` | `period` is outside `202401`–`210012` |

See [ERRORS.md](../ERRORS.md) for the full error catalogue.
