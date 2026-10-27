<div align="center">

# 🔏 HackMe Chain Checkpoints

**Public, signed, Bitcoin-timestamped snapshots of the HackMe blockchain —
so anyone can prove the history was never rewritten.**

[![verify](https://github.com/jokeez/hackme-checkpoints/actions/workflows/verify.yml/badge.svg)](https://github.com/jokeez/hackme-checkpoints/actions/workflows/verify.yml)
[![chain](https://img.shields.io/badge/chain-hackme--dev--mainnet-6f42c1)](https://hackme.tech/pool/explorer/live)
[![signer](https://img.shields.io/badge/signer-HMC--dfc143732a1fc3bc-0a7d5a)](./FORMAT.md#signer)
[![timestamps](https://img.shields.io/badge/anchored_in-Bitcoin_via_OpenTimestamps-f7931a)](https://opentimestamps.org/)

[Verify in 1 minute](#-verify-in-1-minute) · [How it works](#-how-it-works) · [FAQ](#-faq) · [Step-by-step guide](./VERIFY.md) · [Format spec](./FORMAT.md) · [Русский](./README.ru.md)

</div>

---

## 🤔 Why this exists

Today every HackMe block is produced by **one node**. That keeps the network simple, but it
also means you have to *trust* the operator not to quietly rewrite the past — change an old
block, a balance, or how much was minted.

This repository removes that trust **for the past**:

1. Once a day the node takes a **checkpoint**: the hash of the newest block **plus a fingerprint
   of every balance** on the chain.
2. It **signs** the checkpoint with the same key that signs blocks and **publishes** it here.
3. The checkpoint's hash is **anchored in Bitcoin** with OpenTimestamps.

After that, nobody — including the operator — can change history before that point without
it being visible: the live chain would no longer match this repo, and the Bitcoin timestamp
proves when each checkpoint existed.

> **In one sentence:** this repo is a public, append-only receipt book for the HackMe chain.

## ⚡ Verify in 1 minute

You need [Go](https://go.dev/dl/) 1.26+ and git.

```bash
git clone https://github.com/jokeez/hackme-checkpoints
git clone https://github.com/jokeez/hackme
cd hackme
go run ./cmd/checkpoint verify \
  -dir ../hackme-checkpoints/checkpoints \
  -signer HMC-dfc143732a1fc3bc \
  -api https://hackme.tech/pool/api
```

What a good result looks like (numbers will differ):

```text
ok: 12 checkpoints, heights 493901..505000, signer HMC-dfc143732a1fc3bc, chain hackme-dev-mainnet
ok: 12 block hash(es) match https://hackme.tech/pool/api
```

What a **problem** would look like:

```text
checkpoint: HISTORY CHANGED at height 499000: checkpoint 47bd…, API now serves 9c1e…
```

If you ever see that, please [open an issue](https://github.com/jokeez/hackme/issues) —
it means the live chain no longer matches what was published.

Don't want to run anything? The **verify** badge above is GitHub re-checking this repo on every
change and once a day against the live chain, in public.

## 🧩 How it works

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

## 📁 What's in this repo

```text
checkpoints/
  0000493901.json        ← one checkpoint, named by block height
  0000493901.json.ots    ← its OpenTimestamps proof
  …
VERIFY.md                ← step-by-step verification, incl. Bitcoin timestamps
FORMAT.md                ← exact file format, for writing your own verifier
```

A checkpoint is plain JSON you can read in the browser:

```json
{
  "height": 493901,
  "block_hash": "47bd3b85273599755ce2b446ba37c907b29b76c9f7cfe6e776b3e78c9da29ace",
  "state_hash": "ba813b39eae93a179df9b1c3e7d02ec583551fc34fe9360fc057703efacf2f6d",
  "accounts": 260,
  "escrows": 56,
  "totals": { "econ_total_minted_units": 5492067000000, "...": 0 },
  "prev_checkpoint_hash": "",
  "signer_address": "HMC-dfc143732a1fc3bc",
  "sig_ed25519": "8d118132…"
}
```

Amounts are in **units**: `1 HMC = 100,000,000 units`.

## ❓ FAQ

**Does this make HackMe decentralized?**
No. One node still produces blocks. Checkpoints make the *past* tamper-evident; they don't
change who produces the *future*. That is planned separately (see the economics v2 RFC).

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
verifier refuses checkpoints from any other key.

## 🔗 Links

- Main project: [jokeez/hackme](https://github.com/jokeez/hackme) · site: [hackme.tech](https://hackme.tech)
- Live explorer: [hackme.tech/pool/explorer/live](https://hackme.tech/pool/explorer/live)
- Checkpoint code: [`cmd/checkpoint`](https://github.com/jokeez/hackme/tree/main/cmd/checkpoint), [`internal/checkpoint`](https://github.com/jokeez/hackme/tree/main/internal/checkpoint)

<sub>Checkpoint data in this repository is dedicated to the public domain (CC0-1.0).</sub>
