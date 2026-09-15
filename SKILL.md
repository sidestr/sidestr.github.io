---
name: sidestr
description: Make, run or validate a sidestr chain: a user activated sidechain beside Bitcoin or its BLAKE2b fork. Signed blocks, no subsidy, pegged coins with a refund path, rules as documents. A chain is a JSON file and a key.
---

# sidestr for agents

A sidestr chain runs Bitcoin's transaction rules beside a parent chain. Blocks are valid because
they carry a signature satisfying the chain's **challenge** (BIP 325's shape), the subsidy is
zero, and every coin is a coin locked in a **peg** on the parent, refundable to the pegger after
a timelock with nobody's permission. **Signers order blocks; users enforce the rules**: a rule is
a signed document with an activation height that each node adopts or refuses.

Read first: https://sidestr.com/spec/ (draft 0.0.1). Source and reference implementation:
https://github.com/sidestr/spec, directory `siding/`. The sibling protocol for mining pools is
https://datstr.com/ (same engine, same documents).

## What you need

- Node.js 24+.
- Checkouts of the engine `bitcoin-desktop/schema` (`SCHEMA`, default `~/bitcoin-desktop/schema`)
  and the node `bitcoin-blake/blaketestnode` (`BLAKETESTNODE`, default
  `~/remote/github.com/bitcoin-blake/blaketestnode`).
- No parent-chain node is needed to run or validate a siding. Making a peg-in needs a wallet on
  the parent.
- Keys are 32-byte hex files, mode 0600, never on a command line.

## Validate an existing chain (no key needed)

    git clone https://github.com/sidestr/spec && cd spec/siding
    node bin/siding.mjs sync --url <producer or mirror base URL> --chain chains/<name>.json --dir ~/.sidestr/<name>

Fetches `blocks.json` and `blocks.dat` (blaketestnode's block file format: `[u32 height][u32
size][block]`), validates every block with the engine, and prints the height it agrees to or
the exact height and rule it stopped at. A mirror is any static host serving those two files
with Range requests. Chain documents for the existing chains are in `siding/chains/`; the first
chain's is `siding/chain.json`.

## Make a chain

1. Write a chain document: `id`, `name`, `parent`, `addressPrefix` (distinct per chain), `magic`,
   `powLimit` (trivial), `refundBlocks`, `pegConfirmations`, `genesisTime`, and `pegs`: the
   parent outpoints, amounts in sats, and the sidechain script each mints to.
2. `node bin/siding.mjs key --create --chain <doc>` makes the signer key at `~/.sidestr/<name>.key`
   and prints the challenge (`5120` + pubkey). Put it in the document as `challenge`; a level 1
   chain mints the pegs to the same script.
3. `node bin/siding.mjs genesis --chain <doc> --dir <dir>` writes block 0. It is deterministic:
   the same document gives the same block, byte for byte. Pin its hash in the document as
   `genesisHash`.
4. A peg-in on the parent (SPEC 6) is a taproot output, key path the peg holder, script path
   `and_v(v:pk(refund),older(refundBlocks))`, plus an `OP_RETURN` marker. Make it before genesis
   so the document can name it.

## Produce blocks

    node bin/siding.mjs produce --chain <doc> --dir <dir> --port 3450 --interval 3600 --tx-interval 10

Blocks are receipts for transactions: one arrives `--tx-interval` seconds after a transaction,
otherwise every `--interval` seconds as a heartbeat. HTTP on the port: `/status.json`, `/tip`,
`/chain.json`, `/blocks.json`, `/blocks.dat` (Range), `/coins/<scriptPubKey hex>`, and
`POST /tx` with a raw transaction in hex. Bind it publicly or put it behind a proxy to let
others submit; by default it listens on localhost.

## Spend

    node bin/siding.mjs send --url http://127.0.0.1:3450 --chain <doc> --to <script hex or address> --amount <sats>

Spends the signer's own coins with a Taproot key-path signature under the chain's unified
sighash. Any wallet that can sign a Taproot input for the engine can do the same; the address
prefix is the chain's `addressPrefix`.

## Levels and honesty

- Level 1 (all chains today): one signer, who is also the peg holder. Validators check every
  rule and trust the signer for ordering and for which pegs exist. For coins with no value.
- Level 2 adds a parent view so peg-in claims are verified. Level 3 adds several signers with
  rotation and recovery. Neither is implemented yet; peg-in claims after genesis and peg-out are
  specified (SPEC 6, 7) and not yet in code.
- A parent may be a siding (SPEC 3.1). The parent's blocks are the child's clock; trust compounds
  with depth.
- Never call anything on a siding by the parent's coin name unless it is backed by a peg. Issued
  test assets say they are unbacked.
