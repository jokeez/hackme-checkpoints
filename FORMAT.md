# Checkpoint format (version 1)

[← back to README](./README.md)

Enough detail to write an independent verifier in any language.

---

## File

`checkpoints/<HEIGHT>.json`, where `<HEIGHT>` is the decimal height zero-padded to **10 digits**.

Examples:

- height `495097` → `checkpoints/0000495097.json`
- height `497672` → `checkpoints/0000497672.json`

Pretty-printed JSON. The file name must equal the `height` inside.
`checkpoints/<HEIGHT>.json.ots` next to it is the OpenTimestamps proof of the file's **exact bytes**.

---

## Fields

| Field | Type | Meaning |
|---|---|---|
| `version` | int | `1` |
| `chain_id` | string | `hackme-dev-mainnet` |
| `height` | uint64 | tip block index at checkpoint time |
| `block_hash` | hex | hash of that block |
| `block_timestamp_unix` | int64 | that block's timestamp |
| `state_hash` | hex | blake3-256 of the state stream (below) |
| `accounts` | int | number of `A` lines in the state stream |
| `escrows` | int | number of `E` lines |
| `totals` | object | the `M` counters, same values as in the stream |
| `prev_checkpoint_hash` | hex or `""` | checkpoint hash of the previous file; `""` only for the first |
| `created_unix` | int64 | when the checkpoint was made |
| `signer_pubkey_ed25519` | hex | 32-byte Ed25519 public key |
| `signer_address` | string | `"HMC-"` + first 16 hex chars of `sha256(pubkey)` |
| `sig_ed25519` | hex | 64-byte signature |

All amounts are integer **units** (`1 HMC = 10^8 units`).

---

## Signed payload

The signature covers the compact JSON (no spaces, Go `encoding/json` style) of all fields
**except** `sig_ed25519`, in exactly this order:

```text
version, chain_id, height, block_hash, block_timestamp_unix, state_hash, accounts, escrows,
totals, prev_checkpoint_hash, created_unix, signer_pubkey_ed25519, signer_address
```

`totals` keys are sorted alphabetically. Re-serialize from the parsed values in this order —
do **not** sign or hash the pretty-printed file bytes.

---

## Checkpoint hash (for `prev_checkpoint_hash`)

blake3-256 (hex) of the compact JSON of **all** fields, same order, with `sig_ed25519` last.

---

## State stream (`state_hash`)

UTF-8 text, blake3-256 over:

```text
hackme-checkpoint-state-v1\n
A|<address>|<hmc_units>|<hmc_nonce>|<sup_units>|<sup_nonce>|<hms_units>|<hms_nonce>\n
E|<campaign_id>|<budget>|<runs_pool>|<bounty_pool>|<runs_paid>|<bounty_paid>|<crash_bonus_paid>|<refunded_bounty>|<status>\n
M|<key>|<value>\n
```

- One `A` line per account, sorted by address.
- One `E` line per Dig/Hunt escrow, sorted by `campaign_id`.
- One `M` line per counter, keys sorted.

Counters (`M`):

- `econ_order_escrow_units`
- `econ_total_burned_units`
- `econ_total_minted_units`
- `hms_total_burned_units`
- `hms_total_minted_units`
- `sup_total_minted_units`

The tip and all state lines are read inside one read-only database transaction, so they describe
the same instant.

---

## Signer

<a id="signer"></a>

`HMC-dfc143732a1fc3bc`, public key
`9b02a948b238b4b993bc19cbe4f6f7571682b2024d496f4f293f26cb67ef1683` — the key that signs every
block (`miner_pubkey_ed25519` in block JSON). A change of signer key is a chain-level event and
the verifier rejects it.

---

## Versioning

Any change to fields, ordering or the state stream bumps `version`. Verifiers must reject
unknown versions.
