# 🔏 HackMe Chain Checkpoints

**Public, signed, Bitcoin-timestamped snapshots of the HackMe blockchain —
so anyone can prove the history was never rewritten.**

[![verify](https://github.com/jokeez/hackme-checkpoints/actions/workflows/verify.yml/badge.svg)](https://github.com/jokeez/hackme-checkpoints/actions/workflows/verify.yml)
[![chain](https://img.shields.io/badge/chain-hackme--dev--mainnet-6f42c1)](https://hackme.tech/pool/explorer/live)
[![signer](https://img.shields.io/badge/signer-HMC--dfc143732a1fc3bc-0a7d5a)](./FORMAT.md#signer)
[![timestamps](https://img.shields.io/badge/anchored_in-Bitcoin_via_OpenTimestamps-f7931a)](https://opentimestamps.org/)
[![license](https://img.shields.io/badge/license-CC0--1.0-lightgrey)](./LICENSE)

[Verify in 1 minute](#-verify-in-1-minute) ·
[How it works](#-how-it-works) ·
[FAQ](#-faq) ·
[Step-by-step](./VERIFY.md) ·
[Format](./FORMAT.md)

---

## Why this exists

Today every HackMe block is produced by **one node**. That keeps the network simple, but it
also means you have to *trust* the operator not to quietly rewrite the past — change an old
block, a balance, or how much was minted.

This repository removes that trust **for the past**:

1. Once a day the node takes a **checkpoint**: the hash of the newest block **plus a fingerprint
   of every balance** on the chain.
2. It **signs** the checkpoint with the same key that signs blocks and **publishes** it here.
3. The checkpoint's hash is **anchored in Bitcoin** with [OpenTimestamps](https://opentimestamps.org/).

After that, nobody — including the operator — can change history before that point without
it being visible: the live chain would no longer match this repo, and the Bitcoin timestamp
proves when each checkpoint existed.

> **In one sentence:** this repo is a public, append-only receipt book for the HackMe chain.

This does **not** make HackMe decentralized. It makes the *past* tamper-evident. Who produces
the *future* is a separate question — see [`RFC_ECONOMICS_V2`](https://github.com/jokeez/hackme/blob/main/docs/RFC_ECONOMICS_V2.md).

---

## Verify in 1 minute

You need [Go](https://go.dev/dl/) 1.26+ and `git`.

```bash
git clone https://github.com/jokeez/hackme-checkpoints
git clone https://github.com/jokeez/hackme
cd hackme
go run ./cmd/checkpoint verify \
  -dir ../hackme-checkpoints/checkpoints \
  -signer HMC-dfc143732a1fc3bc \
  -api https://hackme.tech/pool/api
```

What a good result looks like (numbers grow as new checkpoints are published):

```text
ok: 2 checkpoints, heights 495097..497672, signer HMC-dfc143732a1fc3bc, chain hackme-dev-mainnet
ok: 2 block hash(es) match https://hackme.tech/pool/api
```

What a **problem** would look like:

```text
checkpoint: HISTORY CHANGED at height 497672: checkpoint 5bd1…, API now serves 9c1e…
```

If you ever see that, please [open an issue](https://github.com/jokeez/hackme/issues) —
it means the live chain no longer matches what was published.

Don't want to run anything? The **verify** badge above is GitHub re-checking this repo on every
push and once a day against the live chain, in public.

For Bitcoin timestamps and a slower walkthrough, see [VERIFY.md](./VERIFY.md).

---

## How it works

```text
                ┌──────────────────────────── once a day ────────────────────────────┐
  HackMe node   │  read tip block + ALL balances  ──►  checkpoint JSON  ──►  sign     │
                └──────────────────────────────────────────────────────┬──────────────┘
                                                                       │
                      ┌────────────────────────────────────────────────┼──────────────┐
                      ▼                                                ▼              │
          this git repo (public history)              OpenTimestamps → Bitcoin block  │
                      │                                                               │
                      └──── each checkpoint also contains the hash of the previous ───┘
                            one, so they form a chain of their own
```

| What a checkpoint pins down | Why it matters |
|---|---|
| **Block hash at a height** | Old blocks can't be replaced. |
| **State fingerprint** (`state_hash`) | Every account's HMC / SUP / HMS balance and nonce, every Dig/Hunt escrow, and the mint/burn counters — balances can't be silently edited. |
| **Supply counters** (`totals`) | How much HMC was minted and burned, readable at a glance. |
| **Link to the previous checkpoint** | Deleting or swapping an old checkpoint breaks every newer one. |
| **Signature** | Only the block-producing node key can create valid checkpoints. |
| **Bitcoin timestamp** (`.ots`) | Proves the checkpoint existed at that time — it can't be back-dated. |

---

## What's in this repo

```text
checkpoints/
  0000495097.json        ← one checkpoint, named by zero-padded height
  0000495097.json.ots    ← its OpenTimestamps proof
  0000497672.json
  0000497672.json.ots
  …
VERIFY.md                ← step-by-step verification, incl. Bitcoin timestamps
FORMAT.md                ← exact file format, for writing your own verifier
```

A checkpoint is plain JSON you can read in the browser. Latest shape (field values change):

```json
{
  "version": 1,
  "chain_id": "hackme-dev-mainnet",
  "height": 497672,
  "block_hash": "5bd1986a510073a2004303a7eb50e38b10c3362b3707f65f9eda69761c092224",
  "state_hash": "64c8e7d6361430043e56b05ec8c152ac1935da92d31a2f90885539184ef914d7",
  "accounts": 260,
  "escrows": 56,
  "totals": {
    "econ_total_minted_units": 5495838000000,
    "econ_total_burned_units": 1304415200
  },
  "prev_checkpoint_hash": "246aa3a5139d9f4020262f5e28d34c5349f41380ee80e0839284f914418feaf0",
  "signer_address": "HMC-dfc143732a1fc3bc",
  "sig_ed25519": "49c8481e…"
}
```

Amounts are in **units**: `1 HMC = 100,000,000 units`. Full field list: [FORMAT.md](./FORMAT.md).

---

## FAQ

**Does this make HackMe decentralized?**
No. One node still produces blocks. Checkpoints make the *past* tamper-evident; they don't
change who produces the *future*. That is planned separately — see
[`RFC_ECONOMICS_V2`](https://github.com/jokeez/hackme/blob/main/docs/RFC_ECONOMICS_V2.md).

**Can the operator just stop publishing?**
Yes — and that is visible too: a missing daily checkpoint is itself a signal. Everything
already published stays verifiable forever.

**Can the operator edit or delete a published checkpoint?**
Not without detection. Every newer checkpoint links to the older one's hash, anyone who
cloned this repo keeps the original, and the Bitcoin timestamp proves what existed when.

**Can I check `state_hash` myself?**
Not from the public API alone — it needs the full ledger. It's a binding commitment: if there
is ever a dispute about balances at a checkpointed height, the answer must reproduce this
exact hash. Block hashes, signatures and links you *can* check yourself today.

**Why OpenTimestamps and not a token on another chain?**
It's free, open, needs no account or coin, and its proofs are verifiable against Bitcoin by
anyone forever.

**Who signs?**
The node key `HMC-dfc143732a1fc3bc` — the same key whose signature is on every block. The
verifier refuses checkpoints from any other key. Public key:
`9b02a948b238b4b993bc19cbe4f6f7571682b2024d496f4f293f26cb67ef1683`.

---

## Links

- Main project: [jokeez/hackme](https://github.com/jokeez/hackme) · site: [hackme.tech](https://hackme.tech)
- Live explorer: [hackme.tech/pool/explorer/live](https://hackme.tech/pool/explorer/live)
- Public API: [`https://hackme.tech/pool/api`](https://hackme.tech/pool/api)
- Checkpoint code: [`cmd/checkpoint`](https://github.com/jokeez/hackme/tree/main/cmd/checkpoint),
  [`internal/checkpoint`](https://github.com/jokeez/hackme/tree/main/internal/checkpoint)
- Economics RFC: [`docs/RFC_ECONOMICS_V2.md`](https://github.com/jokeez/hackme/blob/main/docs/RFC_ECONOMICS_V2.md)

Checkpoint data and docs in this repository are dedicated to the public domain ([CC0-1.0](./LICENSE)).
