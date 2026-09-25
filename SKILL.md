---
name: sidestr
description: Use, make, run or validate a sidestr chain: a user-activated sidechain beside Bitcoin (stock testnet4 or the BLAKE2b fork). Signed blocks, no subsidy, pegged coins, optional rules (assets and a pool, an EVM). One npm package, one command, keys in files; every client validates every block itself.
---

# sidestr for agents

A sidestr chain runs Bitcoin's transaction rules beside a parent chain. Blocks are valid because
they carry a signature satisfying the chain's **challenge**; the subsidy is zero; every coin is
backed by a **peg** on the parent. **Signers order blocks; users enforce the rules.** Nothing is
trusted from a server: a client finds the chain from a signed announcement on public Nostr
relays, reads its blocks from a static mirror, and validates every one itself.

Read: the spec at https://sidestr.com/spec/ (draft 0.0.4; sections 3 chain, 6 peg-in, 7 peg-out,
9 levels, 11 distribution, 12 assets, appendix A event kinds). Source and reference
implementation: https://github.com/sidestr/spec (`siding/`). Client library and command:
https://github.com/sidestr/sidestr. Sibling protocol for mining pools: https://datstr.com/.

## Use a chain (most agents want only this)

    npm install -g sidestr        # Node 22+; the engine, the library and the explorer come with it

A key is 32 bytes of hex in a file, mode 0600, never on a command line; a Nostr secret is one.

    node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" > me.key && chmod 600 me.key
    sidestr info    --chain sidestr:melchain                     # the chain, its tip, the mirror judged against the announcement
    sidestr whoami  --chain sidestr:melchain --key-file me.key   # address, pubkey, did:nostr
    sidestr faucet  --chain sidestr:melchain --key-file me.key --wait   # 100,000 sats where a faucet runs (one per address per day)
    sidestr balance | coins | history [addr]                     # of the key's address, or of addr (no key needed)
    sidestr send <address> <sats> --key-file me.key --yes --wait # prints the plan; --yes broadcasts; --wait until mined (~10 s)
    sidestr data "<text>" --key-file me.key --yes                # an OP_RETURN note, up to 80 bytes
    sidestr sign <txid> --key-file me.key                        # Schnorr signature over 32 bytes: prove a payment was yours
    sidestr tx <txid>                                            # the block it was mined in

`--json` on any command; exit 0 ok, 1 error, 2 not enough funds. `--chain <id>` finds the mirror
from the signer's announcement and remembers it; `--mirror <url>` pins one. Validated state is
cached in `~/.cache/sidestr`, so a later run checks only new blocks. In JavaScript:

    import { openWallet } from 'sidestr';
    const w = await openWallet({ chain: 'sidestr:melchain' });   // every block validated here
    const me = w.identity(key); w.balance(me.script); w.coins(me.script); w.history(me.script);
    const b = w.build({ key, to, amount }); w.verify(b, key); await w.publish(b.hex); await w.mined(b.txid);

Chains that run today (find them all in the directory, https://play-grounds.github.io/sidestr/):
`sidestr:txbt4-siding` (the first), `sidestr:melchain`, `sidestr:capewars-s6`, `sidestr:gitmark`,
`sidestr:txbt4-desk`, `sidestr:txbt4-fed` (level 2, three signers), `sidestr:tally` (rules
`assets`, `pool`: an issued asset SHELL and a SHELL/sats pool), `sidestr:txbt4-evm` (rule `evm`:
Ethereum contracts, 1 sat = 1 gwei, ERC-20 tokens), and `sidestr:dreamlab` beside stock testnet4.
Faucets run on siding, melchain, gitmark, capewars-s6, tally and the evm chain. Test coins, no value.

Examples in the package: `examples/agent.mjs` (an agent in forty lines), `examples/paywall/`
(a service that opens a route to whoever paid it 0.01 SHELL and signs the txid with the key that
paid: it verifies the chain itself, no account, no oracle).

Pages for people: wallet https://sidestr.com/wallet/?chain=<id>, explorer
https://sidestr.com/explorer/?chain=<id>, and in the play-grounds a phone wallet, a station (one
key across every chain, the bridge to and from the parent) and an exchange over the tally pool.

## How it is distributed (SPEC 11)

- A producer announces its chain tip as a signed Nostr event, kind 33333, `d` = chain id: height,
  the last twelve headers, mirror URLs, and since 0.0.4 the parent script a peg-in pays. A chain
  id is a name, not a proof: clients remember the signer they settled on.
- Transactions travel as kind 23500 events (content the hex, tagged `chain`); a faucet answers
  kind 23501 (content an address); a signed *parent* transaction goes as kind 23503 and a producer
  with a node broadcasts it only if its node's own policy accepts it. Relays: nos.lol,
  relay.primal.net, nostr.mom, nostr.oxtr.dev. Relays cap connections per address: share sockets.
- A mirror is any static host serving `chain.json`, `blocks.json` and `blocks.dat` (Range
  requests, CORS). Clients judge a mirror against the announcement; it may lag, never lead.

## Pegs (SPEC 6, 7)

- **Peg-in:** a parent transaction paying the chain's announced peg script, plus an `OP_RETURN`
  of `pegin:<chain id>:<your 34-byte script>`. Claimed after 6 parent confirmations as a coinbase
  on the chain, spendable 100 chain blocks later. The wallet library builds one from a browser key
  (`buildPegIn`, `publishParent`); the station's bridge does it in a page.
- **Peg-out:** a chain transaction burning at least 10,000 sats to `OP_RETURN pegout:<parent
  script>`; the peg holders pay it on the parent within the parent's next blocks
  (`sidestr send --pegout` in the reference command; the wallet's `build({ pegout: true })`).
- Level 1: one signer holds the peg. Level 2: k of n signers, the peg a `tr(…, multi_a(k, …))`
  output, blocks and peg-outs made by a round of events (kinds 23510–23514). Trust is the
  signers; validators check everything else.

## Make and run a chain (operators)

    git clone https://github.com/sidestr/spec && cd spec/siding && npm install
    node bin/siding.mjs new --name <name> --prefix <hrp> [--parent txbt4|xbt|tbtc4|btc] [--rules assets,pool]
    node bin/siding.mjs produce --chain chains/<name>/chain.json --dir ~/.sidestr/<name> --port 3450 \
      --interval 3600 --tx-interval 10 --relay wss://nos.lol,wss://relay.primal.net,wss://nostr.mom,wss://nostr.oxtr.dev \
      --parent-rpc http://127.0.0.1:<port>/ --parent-cookie <node cookie> --parent-wallet <name>-peg \
      --announce-mirror https://<host>/<path>

`new` writes the document, the signer key (`~/.sidestr/<name>.key`) and the genesis, and prints
the lines to run and mirror it. `--parent-rpc` needs a node on the parent (Knots for the BLAKE2b
chains, Core for stock testnet4); `--parent-wallet` is the peg wallet that claims peg-ins and pays
peg-outs; the producer announces one labelled address of it as the peg script. A level 2 chain:
`new … --signers pk1,pk2,pk3 --threshold 2 --key-files …`, then `peg-wallet` per signer. A
chain's header format and signature rule follow the parent (SPEC 3): v2 BLAKE2b headers and the
unified sighash beside `txbt4`/`xbt`, stock headers and BIP 341 beside `tbtc4`/`btc`.

`node bin/siding.mjs sync --url <mirror> --dir <dir>` validates a chain from a mirror and prints
the height it agrees to or the height and rule it stopped at.

## Rules of the estate

- Keys in files only; never in a command line, a URL, or a message.
- Mine nothing a public node refuses; a producer broadcasts a parent transaction only if its own
  node's default policy accepts it.
- Coins are testnet coins. Issued assets say they are unbacked. Never call a sidechain coin by the
  parent's name unless a peg backs it.
- One block per hour when idle, ten seconds after a transaction. Be gentle with mirrors and relays.
