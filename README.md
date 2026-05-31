![Base](logo.webp)

# Base Node

Base is a secure, low-cost, developer-friendly Ethereum L2 built on Optimism's [OP Stack](https://docs.optimism.io/). This repository contains Docker builds to run your own node on the Base network.

[![Website base.org](https://img.shields.io/website-up-down-green-red/https/base.org.svg)](https://base.org)
[![Docs](https://img.shields.io/badge/docs-up-green)](https://docs.base.org/)
[![Discord](https://img.shields.io/discord/1067165013397213286?label=discord)](https://base.org/discord)
[![Twitter Base](https://img.shields.io/twitter/follow/Base?style=social)](https://x.com/Base)
[![Farcaster Base](https://img.shields.io/badge/Farcaster_Base-3d8fcc)](https://farcaster.xyz/base)

## Quick Start

1. Ensure you have an Ethereum L1 full node RPC available
2. Choose your network:
   - For mainnet: Use `.env.mainnet`
   - For testnet: Use `.env.sepolia`
3. Configure your L1 endpoints in the appropriate `.env` file:
   ```bash
   OP_NODE_L1_ETH_RPC=<your-preferred-l1-rpc>
   OP_NODE_L1_BEACON=<your-preferred-l1-beacon>
   OP_NODE_L1_BEACON_ARCHIVER=<your-preferred-l1-beacon-archiver>
   ```
4. Start the node:

   ```bash
   # For mainnet (default):
   docker compose up --build

   # For testnet:
   NETWORK_ENV=.env.sepolia docker compose up --build

   # To use a specific client (optional):
   CLIENT=reth docker compose up --build

   # For testnet with a specific client:
   NETWORK_ENV=.env.sepolia CLIENT=reth docker compose up --build
   ```

### Supported Clients

- `reth` (default)
- `geth`
- `nethermind`

## Requirements

### Minimum Requirements

- Modern Multicore CPU
- 32GB RAM (64GB Recommended)
- NVMe SSD drive
- Storage: (2 \* [current chain size](https://base.org/stats) + [snapshot size](https://basechaindata.vercel.app) + 20% buffer) (to accommodate future growth)
- Docker and Docker Compose

### Production Hardware Specifications

The following are the hardware specifications we use in production:

#### Reth Archive Node (recommended)

- **Instance**: AWS i7i.12xlarge
- **Storage**: RAID 0 of all local NVMe drives (`/dev/nvme*`)
- **Filesystem**: ext4

#### Geth Full Node

- **Instance**: AWS i7i.12xlarge
- **Storage**: RAID 0 of all local NVMe drives (`/dev/nvme*`)
- **Filesystem**: ext4

> [!NOTE]
To run the node using a supported client, you can use the following command:
`CLIENT=supported_client docker compose up --build`

Supported clients:
 - reth (runs vanilla node by default, Flashblocks mode enabled by providing RETH_FB_WEBSOCKET_URL, see [Reth Node README](./reth/README.md))
 - geth
 - nethermind

## Configuration

### Required Settings

- L1 Configuration:
  - `OP_NODE_L1_ETH_RPC`: Your Ethereum L1 node RPC endpoint
  - `OP_NODE_L1_BEACON`: Your L1 beacon node endpoint
  - `OP_NODE_L1_BEACON_ARCHIVER`: Your L1 beacon archiver endpoint
  - `OP_NODE_L1_RPC_KIND`: The type of RPC provider being used (default: "debug_geth"). Supported values:
    - `alchemy`: Alchemy RPC provider
    - `quicknode`: QuickNode RPC provider
    - `infura`: Infura RPC provider
    - `parity`: Parity RPC provider
    - `nethermind`: Nethermind RPC provider
    - `debug_geth`: Debug Geth RPC provider
    - `erigon`: Erigon RPC provider
    - `basic`: Basic RPC provider (standard receipt fetching only)
    - `any`: Any available RPC method
    - `standard`: Standard RPC methods including newer optimized methods

### Network Settings

- Mainnet:
  - `RETH_CHAIN=base`
  - `OP_NODE_NETWORK=base-mainnet`
  - Sequencer: `https://mainnet-sequencer.base.org`

### Performance Settings

- Cache Settings:
  - `GETH_CACHE="20480"` (20GB)
  - `GETH_CACHE_DATABASE="20"` (4GB)
  - `GETH_CACHE_GC="12"`
  - `GETH_CACHE_SNAPSHOT="24"`
  - `GETH_CACHE_TRIE="44"`

### Optional Features

- EthStats Monitoring (uncomment to enable)
- Trusted RPC Mode (uncomment to enable)
- Snap Sync (experimental)

For full configuration options, see the `.env.mainnet` file.

## Snapshots

Snapshots are available to help you sync your node more quickly. See [docs.base.org](https://docs.base.org/chain/run-a-base-node#snapshots) for links and more details on how to restore from a snapshot.

## Supported Networks

| Network | Status |
| ------- | ------ |
| Mainnet | ✅     |
| Testnet | ✅     |

## Troubleshooting

This section covers common issues when setting up and running a Base node. For more detailed guides, see the [Troubleshooting guide on docs.base.org](https://docs.base.org/base-chain/node-operators/troubleshooting).

### Node Won't Start

- **Check Docker is running**: Ensure the Docker daemon is running (`docker info`).
- **Verify environment config**: Confirm L1 endpoints are set correctly in `.env.mainnet` or `.env.sepolia`. Missing or incorrect `OP_NODE_L1_ETH_RPC` or `OP_NODE_L1_BEACON` will cause startup failures.
- **Check port availability**: Ensure ports `8545`, `8546`, `8551`, `6060`, `7545`, and `30303` are not in use by other services:
  ```bash
  sudo lsof -i -P -n | grep LISTEN
  ```
- **JWT secret errors**: If you see authentication errors between `op-node` and the execution client, ensure `OP_NODE_L2_ENGINE_AUTH` is correctly set. The docker-compose setup handles this automatically — don't manually modify unless instructed.
- **Permission issues**: If using `sudo` with Docker, you may encounter permission issues with data directories. Try running as a non-root user added to the `docker` group.

### Syncing Is Stuck

- **Check L1 node health**: Your L1 node must be fully synced and accessible. Check your `OP_NODE_L1_ETH_RPC` is responding:
  ```bash
  curl -d '{"id":0,"jsonrpc":"2.0","method":"eth_blockNumber","params":[]}' \
    -H "Content-Type: application/json" <your-l1-rpc-url>
  ```
- **Monitor sync progress**: Check `optimism_syncStatus` on port `7545`:
  ```bash
  curl -d '{"id":0,"jsonrpc":"2.0","method":"optimism_syncStatus"}' \
    -H "Content-Type: application/json" http://localhost:7545
  ```
- **Check system time**: Ensure the server clock is synchronized using `ntp` or `chrony`. Time drift can cause P2P sync issues.
- **Review logs**: Check both containers for errors:
  ```bash
  docker compose logs -f execution
  docker compose logs -f node
  ```

### Consensus Client Fails

- **Reth fails to start**: Check `RETH_CHAIN` matches your network (`base` or `base-sepolia`). Verify `RETH_SEQUENCER_HTTP` is set correctly.
- **Engine API connection errors**: Ensure JWT authentication is working. The `op-node` connects to the execution client on port `8551` via Engine API.
- **Out of memory**: Reth requires significant RAM. Ensure you meet the [minimum requirements](#requirements) — 32GB RAM minimum, 64GB recommended.
- **Database corruption**: If Reth fails with database errors, you may need to re-sync from a snapshot. See [Snapshots](#snapshots).

### P2P / Networking Issues

- **Node has no peers**: Ensure your firewall allows:
  - **Ingress**: TCP/UDP on ports `30303` and `9222`
  - **Egress**: TCP/UDP on ports `30301`, `30303`, `9200`, and `9222`
  If outbound to bootnodes is blocked, your node cannot discover peers.
- **Behind NAT**: If the node is behind NAT, configure external IP via `ADDITIONAL_ARGS` in your `.env`:
  ```
  ADDITIONAL_ARGS="--nat=extip:<your-external-ip>"
  ```
- **Low peer count**: Check logs for P2P errors. If peers are low, verify all required ports are open in both directions.

### Snapshot Restore Fails

- **Download corruption**: Verify the download URL and retry. Check available disk space — extraction requires significantly more space than the download.
- **Wrong data location**: After extraction, ensure chain data is directly in `./reth-data/`, not in a nested subfolder (e.g., not `./reth-data/reth/db/`).
- **Container not stopped**: Always run `docker compose down` before manually modifying data directories.

### Getting Help

For additional support:
- **Discord**: Join the [Base Discord](https://discord.gg/buildonbase) and post in `🛠｜node-operators`
- **GitHub Issues**: [Open an issue](https://github.com/base/node/issues) if you suspect a bug

## Disclaimer

THE NODE SOFTWARE IS PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND. We make no guarantees about asset protection or security. Usage is subject to applicable laws and regulations.

For more information, visit [docs.base.org](https://docs.base.org/).


## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request
