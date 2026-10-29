# Verifying HackMe checkpoints, step by step

[← back to README](./README.md)

There are three independent things you can check. Each one catches a different kind of tampering.

| # | Check | Catches | Needs |
|---|---|---|---|
| 1 | Signatures and links | forged, edited, deleted or reordered checkpoints | Go |
| 2 | Block hashes vs the live chain | old blocks replaced after publication | Go + internet |
| 3 | Bitcoin timestamps | checkpoints made up later and back-dated | `ots` client |

---

## 0. Get the code

```bash
git clone https://github.com/jokeez/hackme-checkpoints
git clone https://github.com/jokeez/hackme
cd hackme
```

The verifier is small and readable:
[`cmd/checkpoint/main.go`](https://github.com/jokeez/hackme/blob/main/cmd/checkpoint/main.go)
and
[`internal/checkpoint/checkpoint.go`](https://github.com/jokeez/hackme/blob/main/internal/checkpoint/checkpoint.go).
It reads only the files in `checkpoints/` and, for step 2, public HTTPS.

---

## 1. Signatures and links (offline)

```bash
go run ./cmd/checkpoint verify \
  -dir ../hackme-checkpoints/checkpoints \
  -signer HMC-dfc143732a1fc3bc
```

For every checkpoint, in height order, this checks:

- the Ed25519 signature over the exact signed payload;
- `signer_address` really belongs to `signer_pubkey_ed25519`, and it is `HMC-dfc143732a1fc3bc`;
- every checkpoint is signed by the **same** key;
- heights strictly increase;
- `prev_checkpoint_hash` equals the hash of the previous checkpoint (so nothing was removed,
  inserted or replaced in between). The first checkpoint may have an empty
  `prev_checkpoint_hash`.

Expected output shape:

```text
ok: 2 checkpoints, heights 495097..497672, signer HMC-dfc143732a1fc3bc, chain hackme-dev-mainnet
```

---

## 2. Block hashes against the live chain

```bash
go run ./cmd/checkpoint verify \
  -dir ../hackme-checkpoints/checkpoints \
  -signer HMC-dfc143732a1fc3bc \
  -api https://hackme.tech/pool/api
```

For each checkpoint it asks the public API which block is at that height and compares the hash.
It also checks the block's miner is the checkpoint signer. Add `-last 30` to only check the
newest 30.

You can do one by hand in a browser — replace the height with any file in `checkpoints/`:

`https://hackme.tech/pool/api/reports/blocks?q=497672`

Compare the response `hash` with `block_hash` in `checkpoints/0000497672.json`.

---

## 3. Bitcoin timestamps

```bash
pip install opentimestamps-client
ots info   ../hackme-checkpoints/checkpoints/0000497672.json.ots
ots verify ../hackme-checkpoints/checkpoints/0000497672.json.ots
```

- Right after publishing, proofs may be *pending* (calendar servers promise to include them).
  The node upgrades them in this repo once a Bitcoin block contains them.
- `ots verify` needs access to Bitcoin block headers (a local Bitcoin Core node, or see the
  [OpenTimestamps docs](https://opentimestamps.org/) for alternatives). It prints the Bitcoin
  block and the time the file is proven to have existed by.
- Changing even one byte of the JSON file makes `ots verify` fail.
- Always verify the `.ots` that sits next to the matching `.json` — never a `.bak` or copy.

---

## What the checks do **not** cover

- **`state_hash`** can't be recomputed from the public API — it needs the full ledger. It is a
  binding commitment for any future dispute about balances at that height.
- **Future blocks** — checkpoints protect the past, not who produces the next block.
- **Gaps** — if a day is missing, that is visible but not "invalid"; ask why on the main repo.

---

## Found a problem?

Open an issue on [jokeez/hackme](https://github.com/jokeez/hackme/issues) with the command you ran
and its full output. Please keep your clone — it is evidence.
