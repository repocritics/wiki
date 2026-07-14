# ethereum/go-ethereum

> The reference Go implementation of an Ethereum execution-layer client — the node software known as Geth.

[GitHub repo](https://github.com/ethereum/go-ethereum) ·
[Official website](https://geth.ethereum.org) ·
[License: LGPL-3.0 (library) / GPL-3.0 (binaries)](https://github.com/ethereum/go-ethereum/blob/master/COPYING.LESSER)

## Overview

Go-ethereum ("Geth") is one of the original Ethereum clients, shipping with the Frontier mainnet launch in July 2015[^1]. It is written in Go and provides `geth`, a full node that speaks the Ethereum wire protocol over devp2p, maintains the world state, executes the EVM, and exposes JSON-RPC APIs over HTTP, WebSocket, and IPC. For most of Ethereum's history Geth was *the* node — and it remains the most widely deployed execution client, which is both its strength and the ecosystem's largest single point of concern.

Since **The Merge** (September 2022), Ethereum is no longer a single-process system. Geth is now only the *execution layer* (EL): it processes transactions and state, but no longer decides which block is canonical and no longer mines. Consensus (proof-of-stake, fork choice, block production timing) is handled by a separate *consensus-layer* (CL) client — Prysm, Lighthouse, Teku, Nimbus, or Lodestar — that pairs with Geth over the authenticated **Engine API**[^2]. Running "a node" today means running two processes with a shared JWT secret. This is the single biggest conceptual shift for anyone whose mental model predates 2022.

The defining tension around Geth is **client diversity**. Because a supermajority of Ethereum nodes historically ran Geth, a consensus-affecting bug in Geth could split the chain or stall finality in a way that a minority-client bug could not. The community actively lobbies stakers to run alternative EL clients (Nethermind, Besu, Erigon, Reth) specifically to keep any single implementation below the ~33% and ~66% safety thresholds. Geth's maintainers agree with this goal, which is unusual: the project's own documentation encourages some users to run something else.

## Getting Started

Building from source requires Go 1.23+ and a C compiler[^3]:

```shell
git clone https://github.com/ethereum/go-ethereum
cd go-ethereum
make geth          # or `make all` for the full tool suite
```

Snap-sync a mainnet full node (this alone is *not* a complete node post-Merge — you also need a consensus client):

```shell
geth --http --http.api eth,net,web3 \
     --authrpc.jwtsecret /secrets/jwt.hex \
     console
```

A minimal Go program reading chain state via the `ethclient` package:

```go
package main

import (
	"context"
	"fmt"

	"github.com/ethereum/go-ethereum/ethclient"
)

func main() {
	client, err := ethclient.Dial("http://localhost:8545")
	if err != nil {
		panic(err)
	}
	block, err := client.BlockByNumber(context.Background(), nil) // latest
	if err != nil {
		panic(err)
	}
	fmt.Println("head block:", block.Number().Uint64())
}
```

## Architecture / How It Works

Geth is a monolithic Go binary composed of loosely coupled subsystems:

- **P2P / networking** — devp2p over the RLPx transport, with `discv4`/`discv5` node discovery. The `eth` protocol handles block and transaction propagation; the `snap` protocol serves state snapshots for fast sync.
- **Sync** — the default is **snap sync**: download account/storage snapshots plus recent blocks instead of re-executing all history, then heal the trie in the background. **Full sync** re-executes every block from genesis. An **archive node** (`--gcmode=archive`) additionally retains every historical state root, costing multiple terabytes. Light-client mode (`les`) has been effectively abandoned post-Merge.
- **State & storage** — account state is a Merkle-Patricia Trie. Geth ships two state schemes: the legacy **hash-based** layout and the newer **path-based state scheme (PBSS)**, which prunes automatically and keeps disk growth bounded. The key-value backend moved from LevelDB to **Pebble** (a Go-native RocksDB-alike) as the default[^4].
- **EVM** — a from-scratch Go interpreter for Ethereum bytecode, shared by the node and the standalone `evm` debugging tool.
- **Engine API** — since the Merge, block insertion is driven *externally* by the consensus client via `engine_newPayload` / `engine_forkchoiceUpdated` on an authenticated port (default 8551, JWT-protected). Geth no longer runs its own fork-choice.

Ancillary tooling ships in `cmd/`: `abigen` (generates type-safe Go bindings from Solidity ABIs), `devp2p`, `evm`, and `rlpdump`. The `cmd/` binaries are GPL-3.0; everything outside `cmd/` is LGPL-3.0, which matters if you embed Geth packages as a library.

## Production Notes

- **You must run two clients.** A Geth-only setup will sync the EL but will never advance the head post-Merge — the consensus client drives block insertion. Forgetting the `--authrpc.jwtsecret` (shared with the CL) is the most common first-node failure.
- **Disk is the binding constraint.** A pruned full node needs ~1TB+ of fast NVMe SSD and grows continuously; the chain outpaces slow disks. Archive nodes are multi-terabyte and keep climbing. Consumer QLC SSDs and most network/cloud block storage are too slow for IOPS-bound state access — expect missed attestations if you're staking on underpowered disk.
- **Snap sync is fast but fragile on bad peers/disk.** Interrupted syncs on slow storage can stall in the "state heal" phase for a long time. Full sync is slower but more predictable.
- **State scheme migration is one-way in practice.** Switching an existing hash-based database to PBSS is not a free in-place flip; the supported path is often a resync. Decide the scheme before provisioning.
- **RPC exposure is a real attack surface.** HTTP/WS bind to localhost by default for good reason. Exposing `--http.addr 0.0.0.0` without a firewall, and enabling admin/personal namespaces, has drained nodes. Never expose the `authrpc` port publicly.
- **Mining code is gone.** Post-Merge, `geth` cannot mine or produce PoW blocks; block *building* for validators is delegated to the CL and (usually) external block builders via MEV-Boost. Old tutorials referencing `--mine` or ethash are obsolete.
- **Private networks changed.** Since the Merge you can no longer stand up a multi-node private net with Geth alone — you need a paired beacon chain. For local dev, use Geth's `--dev` mode or the simulated backend; for multi-node, tooling like Kurtosis.

## When to Use / When Not

**Use when:**
- You need a battle-tested, spec-reference execution client and want the implementation most likely to match mainnet behavior exactly.
- You're building Ethereum tooling in Go and want first-class libraries (`ethclient`, `abigen`, `core/vm`).
- You want the broadest documentation, tutorials, and community answers — Geth is the default in most guides.

**Avoid / reconsider when:**
- You're a staker choosing an EL client purely on merit: running a *minority* client (Nethermind, Besu, Erigon, Reth) is the more civic choice for network resilience.
- You need a disk-efficient archive node: Erigon's staged sync stores full history in a fraction of Geth's archive footprint.
- You want a modular EL you can embed as a library / customize deeply: Reth is designed library-first; Geth's internals are stable but not built as a reusable framework.

## Alternatives

- erigontech/erigon — Go EL client with staged sync; dramatically smaller archive nodes, better for indexing/analytics workloads.
- paradigmxyz/reth — Rust EL client, modular and library-first; use when you want to build on or customize the client.
- nethermindeth/nethermind — C#/.NET EL client; a common minority-client choice for stakers seeking diversity.
- hyperledger/besu — Java EL client (Apache-2.0); favored in enterprise/permissioned settings.
- prysmaticlabs/prysm — a *consensus* client, not a Geth alternative but a required companion; pair it (or Lighthouse/Teku/Nimbus) with your EL client.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Frontier | 2015-07-30 | Ethereum mainnet launch; Geth an original client[^1]. |
| Geth 1.10 | 2021-03 | Snap sync, offline pruning, EIP-1559 groundwork. |
| London | 2021-08-05 | EIP-1559 fee market activated on mainnet[^5]. |
| The Merge | 2022-09-15 | PoW/mining removed; external consensus client + Engine API required[^2]. |
| Shapella | 2023-04-12 | Beacon-chain staking withdrawals enabled. |
| Geth 1.13 | 2023-10 | Path-based state scheme (PBSS); Pebble default backend[^4]. |
| Dencun | 2024-03-13 | Blob transactions (EIP-4844 / proto-danksharding) support. |
| Pectra | 2025-05 | Prague/Electra upgrade; EIP-7702, execution-layer requests. |

## References

[^1]: Ethereum Foundation, "Ethereum Launches" (Frontier), 2015-07-30. https://blog.ethereum.org/2015/07/30/ethereum-launches
[^2]: Ethereum.org, "The Merge" and the execution/consensus split. https://ethereum.org/en/roadmap/merge/
[^3]: go-ethereum README, "Building the source" (Go 1.23+ and a C compiler). https://github.com/ethereum/go-ethereum#building-the-source
[^4]: go-ethereum releases and the path-based state storage / Pebble default. https://github.com/ethereum/go-ethereum/releases
[^5]: Ethereum.org, "History and forks — London / EIP-1559". https://ethereum.org/en/history/

## Tags

go, ethereum, blockchain, execution-client, evm, p2p, json-rpc, node, web3, staking
