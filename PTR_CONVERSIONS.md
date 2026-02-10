# PTR Conversions: Script Pubkeys, SPTRs, and P2TR Addresses

## Overview

This document explains the relationships between three representations of Taproot outputs:
- **Script Pubkeys** starting with `5120...` (raw script format)
- **SPTRs** (Space Pointers) starting with `sptr1...` (bech32m encoded identifiers)
- **P2TR Addresses** starting with `bc1p...` (bech32m encoded addresses)

All three represent the same underlying 32-byte Taproot output key, but serve different purposes in the Spaces protocol.

## Format Details

### Script Pubkey (`5120...`)

The script pubkey format for P2TR (Pay-to-Taproot) outputs:
- `51` = `OP_1` (witness version 1)
- `20` = `OP_PUSHDATA1` followed by 32 bytes
- Followed by: 32-byte Taproot output key

**Example:** `51207af118ac81145433ea08f7192b6f69466c73476d14fcae7530e8c5df0b813808`

### SPTR (Space Pointer) (`sptr1...`)

A protocol-specific identifier derived from a script pubkey:
- Format: bech32m encoding with HRP `"sptr"`
- Derived via: `ns_hash(KeyKind::Sptr, hash(script_pubkey_bytes))`
- Result: 32-byte hash encoded as bech32m string

**Example:** `sptr1ur89nczzheactvtgamlpl6hp2ghh3mh6q62km642ql5u3ujyyhrstv3z28`

### P2TR Address (`bc1p...`)

Standard Bitcoin Taproot address:
- Format: bech32m encoding of the 32-byte Taproot output key
- Network prefixes:
  - Mainnet: `bc1p...`
  - Testnet: `tb1p...`
  - Regtest: `bcrt1p...`

**Example:** `bc1pjeda67ewjtms6p20nk3udt6c5wwk90zzdlhd3dx73r8ynzsm07nqwxggmu`

## Conversion Relationships

### 1. Script Pubkey → SPTR

**Direction:** One-way (hash function)

**Process:**
```rust
// From ptr/src/sptr.rs
pub fn from_spk<H: KeyHasher>(spk: ScriptBuf) -> Self {
    Self(ns_hash::<H>(KeyKind::Sptr, H::hash(&spk.as_bytes())))
}
```

**Details:**
- Hash the script pubkey bytes
- Apply namespace hash with `KeyKind::Sptr`
- Encode the resulting 32-byte hash as bech32m with HRP `"sptr"`

**Reversibility:** ❌ Cannot recover script pubkey from SPTR (one-way hash)

**Use Case:** Creating a protocol-specific identifier for a PTR that doesn't expose the underlying script pubkey.

### 2. Script Pubkey ↔ P2TR Address

**Direction:** Bidirectional (encoding/decoding)

**Process:**
- **To Address:** Extract the 32-byte Taproot output key from script pubkey (bytes 2-34), encode as bech32m
- **From Address:** Decode bech32m to get 32-byte key, wrap with `OP_1 OP_PUSHDATA1` prefix

**Reversibility:** ✅ Fully reversible

**Use Case:** Converting between raw script format and human-readable address format.

### 3. P2TR Address → SPTR

**Direction:** Indirect (requires script pubkey as intermediate step)

**Process:**
1. Decode `bc1p` address to get 32-byte Taproot output key
2. Reconstruct script pubkey: `OP_1 OP_PUSHDATA1 + 32-byte key`
3. Hash script pubkey to derive SPTR (as in relationship #1)

**Reversibility:** ❌ Cannot go directly from address to SPTR without script pubkey

**Use Case:** Finding the SPTR associated with a known Bitcoin address.

## Conversion Matrix

| From | To | Method | Reversible |
|------|-----|--------|------------|
| Script Pubkey (`5120...`) | SPTR (`sptr1...`) | Hash + namespace hash | ❌ No |
| Script Pubkey (`5120...`) | P2TR Address (`bc1p...`) | Extract 32 bytes + bech32m encode | ✅ Yes |
| P2TR Address (`bc1p...`) | Script Pubkey (`5120...`) | Decode bech32m + wrap with OP codes | ✅ Yes |
| P2TR Address (`bc1p...`) | SPTR (`sptr1...`) | Via script pubkey (indirect) | ❌ No |
| SPTR (`sptr1...`) | Script Pubkey (`5120...`) | Not possible (one-way hash) | ❌ No |
| SPTR (`sptr1...`) | P2TR Address (`bc1p...`) | Not possible (one-way hash) | ❌ No |

## Implementation Notes

### Creating an SPTR from Script Pubkey

```rust
use spaces_ptr::sptr::Sptr;
use spaces_client::store::Sha256;
use bitcoin::ScriptBuf;

let script_pubkey = ScriptBuf::from_hex("51207af118ac81145433ea08f7192b6f69466c73476d14fcae7530e8c5df0b813808")?;
let sptr = Sptr::from_spk::<Sha256>(script_pubkey);
println!("SPTR: {}", sptr); // e.g., "sptr1ur89nczzheactvtgamlpl6hp2ghh3mh6q62km642ql5u3ujyyhrstv3z28"
```

### Converting Script Pubkey to P2TR Address

```rust
use bitcoin::{Address, Network, ScriptBuf};

let script_pubkey = ScriptBuf::from_hex("51207af118ac81145433ea08f7192b6f69466c73476d14fcae7530e8c5df0b813808")?;
// Extract the 32-byte taproot output key (bytes 2-34)
let taproot_key = &script_pubkey.as_bytes()[2..34];
// Create address (requires additional Bitcoin library functions)
let address = Address::p2tr_tweaked(taproot_key, Network::Bitcoin);
```

### Converting P2TR Address to Script Pubkey

```rust
use bitcoin::{Address, ScriptBuf};

let address: Address = "bc1pjeda67ewjtms6p20nk3udt6c5wwk90zzdlhd3dx73r8ynzsm07nqwxggmu".parse()?;
let script_pubkey = address.script_pubkey();
// script_pubkey will be in format: 5120<32-byte-key>
```

## Use Cases in Spaces Protocol

1. **Creating PTRs:** Users provide script pubkeys (`5120...`) to create PTRs, which are internally tracked by their SPTR identifier.

2. **Address Display:** PTRs can be displayed as either:
   - SPTR format (`sptr1...`) for protocol-specific operations
   - P2TR address format (`bc1p...`) for compatibility with Bitcoin wallets

3. **Lookups:** The protocol uses SPTRs as the primary key for looking up PTR state, while script pubkeys are used for transaction construction.

## Security Considerations

- **SPTR Privacy:** Since SPTRs are one-way hashes, you cannot determine the underlying script pubkey or address from an SPTR alone. This provides some privacy benefits.

- **Address Reuse:** Multiple script pubkeys can theoretically hash to the same SPTR (hash collision), though this is cryptographically infeasible with SHA-256.

- **Key Extraction:** The 32-byte Taproot output key is the common element across all three formats, representing the actual cryptographic key used in Taproot.

## References

- BIP 341: Taproot (SegWit version 1 spending rules)
- BIP 350: Bech32m encoding format
- Code references:
  - `ptr/src/sptr.rs` - SPTR implementation
  - `wallet/src/address.rs` - Address handling
  - `ptr/src/lib.rs` - PTR validation and processing

