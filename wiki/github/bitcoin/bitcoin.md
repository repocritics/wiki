# bitcoin/bitcoin

> Bitcoin Core — the reference full-node implementation of the Bitcoin protocol, and the software whose behavior defines what "the rules" actually are.

[GitHub repo](https://github.com/bitcoin/bitcoin) ·
[Official website](https://bitcoincore.org) ·
[License: MIT](https://github.com/bitcoin/bitcoin/blob/master/COPYING)

## Overview

Bitcoin Core is the direct descendant of the original client Satoshi Nakamoto released in January 2009[^1]. It connects to the Bitcoin peer-to-peer network, downloads and independently validates every block and transaction, maintains the UTXO set, and optionally runs a wallet and a Qt graphical interface. The project was called "Bitcoin" and then "Bitcoin-Qt"; it was renamed "Bitcoin Core" around the 0.9 release (2014) to distinguish the software from the network and the currency.

The defining tension of this repository is that it has no separate specification. There is no formal standards document that Bitcoin Core implements; in practice, whatever Bitcoin Core accepts *is* the consensus rule. This makes every line of consensus-critical code load-bearing in a way most software is not: a change that makes the node accept a block the rest of the network rejects (or reject one they accept) forks the chain and can cost users money directly. That reality shapes everything about the project — the extreme review burden, the conservatism, the reluctance to refactor consensus code, and the near-religious insistence that testing and review, not merge velocity, are the bottleneck[^2].

Bitcoin Core is written in C++ (targeting C++20 as of recent releases) and is used by anyone running a validating node: exchanges, custodians, block explorers, Lightning nodes (as a backend), miners, and individuals who want to verify their own transactions rather than trust a third party. The GUI lives in a separate monotree repository, `bitcoin-core/gui`, whose `master` is kept identical to this tree[^3].

## Getting Started

Most operators use the signed binaries rather than building from source:

```bash
# Download + verify from https://bitcoincore.org/en/download/
# then run a node:
bitcoind -daemon
bitcoin-cli getblockchaininfo
bitcoin-cli stop
```

A minimal `~/.bitcoin/bitcoin.conf` for a pruned node that indexes nothing extra:

```ini
# Keep only ~5 GB of block data instead of the full chain
prune=5000
# Larger UTXO cache dramatically speeds initial sync (MiB)
dbcache=4000
# JSON-RPC — bind to localhost only, never expose to the internet
server=1
rpcbind=127.0.0.1
```

Building from source (CMake-based since the 29.0 cycle; earlier releases used Autotools)[^4]:

```bash
cmake -B build
cmake --build build -j"$(nproc)"
ctest --test-dir build           # unit tests
build/test/functional/test_runner.py   # Python functional tests
```

## Architecture / How It Works

The node is a monolithic C++ process with several tightly coupled subsystems:

- **Validation** (`src/validation.cpp`, `src/consensus/`) — the heart of the system. Connects blocks, applies script and consensus checks, and maintains the coin database. Consensus code is deliberately walled off and changed rarely.
- **Script interpreter** (`src/script/`) — evaluates the stack-based Bitcoin Script for every input, gated by soft-fork flags (SegWit / BIP141, Taproot / BIP341–342).
- **libsecp256k1** — the in-house elliptic-curve library for ECDSA and Schnorr signatures, developed in a sibling repo and vendored here. Replacing OpenSSL for consensus signing removed a large class of cross-platform validation risk.
- **UTXO set** — stored in LevelDB under `chainstate/`; blocks are stored as flat `blk*.dat` files with a LevelDB index. The in-memory coins cache (`dbcache`) is the single biggest lever on sync speed.
- **Mempool + policy** (`src/policy/`) — *policy* rules (minimum fee, RBF, standardness, package relay) are node-local and stricter than *consensus* rules. The distinction matters: a policy change cannot fork the chain; a consensus change can.
- **P2P** (`src/net.cpp`, `src/net_processing.cpp`, `addrman`) — the gossip network, peer management, and block/transaction relay.
- **Wallet** (`src/wallet/`) — optional, descriptor-based since v0.21, backed by SQLite for new wallets. It is a client of the node, not part of consensus.

Two shortcuts make a full validate tractable: **`assumevalid`** skips signature checking (but not structural/UTXO checks) for blocks below a hardcoded recent block hash, and **`assumeutxo`** lets a node bootstrap from a serialized UTXO snapshot and validate the remainder in the background[^5]. Neither weakens the end state — a node still fully validates going forward.

## Production Notes

**Initial Block Download (IBD) is the main operational cost.** A full, unpruned node validates the entire history and stores the whole chain — well over 600 GB by 2026 and growing — plus a chainstate of tens of GB. Depending on hardware and `dbcache`, IBD ranges from several hours (fast NVMe, large cache) to days (spinning disk, small cache). SSD is effectively mandatory; the chainstate does poison-slow random I/O on HDDs.

**Pruning trades history for disk, not security.** `prune=N` discards old block/undo files after validation, cutting storage to a few GB while keeping full validation. The cost: you cannot serve historical blocks to peers, cannot run `txindex`, and rescanning an old wallet is limited.

**Consensus bugs are catastrophic and have happened.** The March 2013 chain split (BIP50) came from a latent Berkeley DB lock limit: v0.7 rejected a large block that v0.8 (LevelDB) accepted, splitting the network for several hours[^6]. CVE-2018-17144, an inflation/double-spend bug from a missing duplicate-input check, shipped for months before being quietly fixed in 0.16.3[^7]. These are the reason the project treats review, not throughput, as the constraint.

**Never expose the RPC port.** The JSON-RPC interface is a full control surface (including the wallet). Bind it to localhost; use `bitcoin-cli` or a reverse proxy with auth. Exposed RPC has drained wallets.

**Wallet migration pain.** Legacy Berkeley DB wallets are deprecated and built-in BDB write support has been removed in recent releases; existing BDB wallets must be migrated to descriptor/SQLite format with the migration tooling before upgrading past that boundary. Budget for this on old deployments.

**Reindexing and upgrades.** A `-reindex` re-validates from disk and can take as long as an IBD. Downgrading across a database format change is generally unsupported — snapshot `chainstate/`/`blocks/` before major upgrades if you may need to roll back.

## When to Use / When Not

**Use when:**
- You need to trust-minimally verify Bitcoin yourself (exchange, custodian, merchant, sovereign individual).
- You are running Lightning, an explorer, or an indexer that needs a validating backend.
- You want the canonical behavior — anything else risks disagreeing with the network on an edge case.

**Avoid / look elsewhere when:**
- You only need to *read* chain data or query balances — a hosted API or a lighter indexer (Electrum server, `electrs`) is cheaper than a full node.
- You want a light client on constrained hardware — SPV/neutrino wallets exist for that; Core is a full node by design.
- You want to embed node logic in a non-C++ service and value ergonomics over being the reference — an alternative implementation may fit better, at the cost of consensus risk.

## Alternatives

- btcsuite/btcd — full-node implementation in Go; use when your stack is Go and you accept that a non-reference node carries consensus-divergence risk.
- bitcoinknots/bitcoin — Bitcoin Knots, a downstream fork of Core with additional features and looser/alternative policy defaults; use when you want Core's consensus code but different node policy.
- bcoin-org/bcoin — full node in JavaScript/Node.js; use when you need to script or embed a node in a JS environment.
- libbitcoin/libbitcoin — modular C++ toolkit for building custom Bitcoin services; use when you want library components rather than a monolithic daemon.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2009-01 | Satoshi's original release; genesis block Jan 3[^1]. |
| 0.3.x | 2010 | Satoshi's last releases before stepping back; OpenSSL-based. |
| 0.9.0 | 2014-03 | "Bitcoin Core" name adopted; move off wxWidgets to Qt matured. |
| 0.10.0 | 2015-02 | libsecp256k1 for verification; headers-first sync. |
| 0.13/0.14 | 2016–17 | SegWit (BIP141) merged; activated on-network Aug 2017[^8]. |
| 0.21.0 | 2021-01 | Descriptor wallets; Tor v3; Schnorr/Taproot code readied. |
| 22.0 | 2021-09 | Dropped the `0.` prefix; Taproot activated Nov 2021[^9]. |
| 24.0 | 2022-12 | Package relay groundwork; full-RBF opt-in. |
| 26.0 | 2023-12 | assumeutxo maturing; migration of legacy wallets. |
| 28.0 | 2024-10 | Continued policy/mempool work; BDB write support wound down. |
| 29.0 | 2025 | CMake build system replaces Autotools[^4]. |

## References

[^1]: Satoshi Nakamoto, "Bitcoin v0.1 released" — 2009-01-09, cryptography mailing list. https://satoshi.nakamotoinstitute.org/emails/cryptography/16/
[^2]: Bitcoin Core README, "Testing" section — review and testing described as the development bottleneck. https://github.com/bitcoin/bitcoin/blob/master/README.md
[^3]: Bitcoin Core README, "Development Process" — the GUI is developed in the `bitcoin-core/gui` monotree. https://github.com/bitcoin-core/gui
[^4]: Bitcoin Core build documentation (CMake). https://github.com/bitcoin/bitcoin/blob/master/doc/build-unix.md
[^5]: Bitcoin Core docs, assumeutxo design notes. https://github.com/bitcoin/bitcoin/blob/master/doc/design/assumeutxo.md
[^6]: BIP50, "March 2013 Chain Fork Post-Mortem." https://github.com/bitcoin/bips/blob/master/bip-0050.mediawiki
[^7]: CVE-2018-17144 disclosure, Bitcoin Core 0.16.3. https://bitcoincore.org/en/2018/09/20/notice/
[^8]: BIP141, "Segregated Witness (Consensus layer)." https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki
[^9]: BIP341, "Taproot: SegWit version 1 spending rules"; activated at block 709632. https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki

## Tags

cpp, bitcoin, cryptocurrency, blockchain, cryptography, p2p, full-node, consensus, wallet, distributed-systems
