# FuelLabs/fuels-ts

The TypeScript SDK for the Fuel Network — a modular blockchain execution layer. Stars-to-watcher ratio warrants review before treating popularity as signal.

## What it is

A TypeScript SDK that abstracts the Fuel Network's RPC + smart-contract interaction surface. Fuel positions itself as a UTXO-based, parallel-execution L2 / modular blockchain; this SDK is the JS/TS-facing way to interact with it. Apache 2.0 licensed.

## Key features

- TypeScript SDK for Fuel Network nodes.
- Smart-contract interaction (Sway language contracts).
- Wallet, signer, and transaction-building primitives.
- npm-distributed.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary.

## When to reach for it

- You're building a dApp on the Fuel Network.
- You're a blockchain developer evaluating UTXO-based execution layers.

## When *not* to reach for it

- You're targeting Ethereum / EVM — use ethers.js or viem instead.
- You're not in the modular-blockchain space.

## Maturity signal

43k stars vs. 108 watchers + ~1.4k forks is an unusual ratio for an SDK; SDKs typically have lower star counts because the audience is project-specific. The watcher-to-star ratio combined with the Fuel ecosystem's specialized niche makes the star count a weak signal of broad utility. Apache 2.0 license clean. Anyone evaluating should verify whether Fuel Network adoption justifies the SDK choice for their project.

## Alternatives

- ethers.js, viem — for EVM chains.
- @solana/web3.js — for Solana.

## Tags

typescript, blockchain, sdk, library, apache-license, fuel
