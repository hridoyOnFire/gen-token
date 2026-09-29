# Agent-to-Agent USDC Transfer

A full-stack demo of **programmatic USDC transfers between AI agents** on [Arc Testnet](https://arc.io) using Circle developer-controlled wallets.

Two wallets (Agent Alpha and Agent Beta) are provisioned via the Circle SDK. A dashboard lets you fire transfers, watch real-time transaction state, and inspect the history — all onchain on Arc, where USDC is the native gas token.

---

## Features

- **Two agent wallets** provisioned in one click via Circle developer-controlled wallets SDK
- **Live USDC balances** polled from the Circle API
- **Programmatic transfers** with real-time state transitions (INITIATED → SENT → COMPLETE)
- **Arc Testnet explorer** links for every confirmed transaction
- **Python CLI** companion (`python/agent_transfer.py`) — run the same flow without a browser
- **Solidity contracts** (DisputeResolver) for USDC escrow-based dispute resolution

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, TypeScript, Tailwind CSS, Framer Motion |
| Backend | Bun + Node HTTP server (TypeScript) |
| Onchain | wagmi v2, viem v2, ConnectKit |
| Wallets | Circle Developer-Controlled Wallets SDK |
| Contracts | Solidity 0.8.28, Foundry, OpenZeppelin 5.1.0 |
| Chain | Arc Testnet (Chain ID: 5042002) |
| Token | USDC (Arc Testnet: `0x3600000000000000000000000000000000000000`) |
| Python CLI | Python 3.11+, httpx, python-dotenv |

---

## Project Structure

```
.
├── src/                          # React frontend
│   ├── App.tsx                   # App root
│   ├── components/
│   │   ├── AgentTransferDashboard.tsx   # Main dashboard
│   │   └── DisputeDashboard.tsx         # Dispute resolver UI
│   ├── onchain-facts.ts          # Chain/USDC addresses (generated)
│   ├── onchain-money.ts          # USDC amount helpers
│   └── onchain-wait.ts           # Transaction state polling
├── server.ts                     # Bun backend — Circle SDK API routes
├── contracts/
│   ├── DisputeResolver.sol       # USDC escrow dispute contract
│   └── test/                     # Foundry tests
├── python/
│   ├── agent_transfer.py         # Python CLI companion
│   ├── requirements.txt
│   └── README.md
├── .github/
│   ├── workflows/ci.yml          # Type-check + lint on push/PR
│   └── ISSUE_TEMPLATE/           # Bug report + feature request templates
├── foundry.toml
└── package.json
```

---

## Quick Start

### Prerequisites

- [Bun](https://bun.sh) v1.0+
- [Foundry](https://getfoundry.sh) (for contract work)
- A [Circle developer account](https://console.circle.com) with an API key + entity secret

### 1. Install dependencies

```bash
bun install
```

### 2. Configure environment

```bash
cp env.example .env
```

Edit `.env`:

```
CIRCLE_DEVELOPER_CONTROLLED_API_KEY=your_api_key_here
CIRCLE_ENTITY_SECRET=your_entity_secret_here
```

### 3. Run the app

```bash
# Terminal 1 — backend
bun server.ts

# Terminal 2 — frontend
bun run dev
```

Open `http://localhost:5173` and click **Provision Agents**.

### 4. Fund Agent Alpha

Use the testnet faucet to send USDC to Agent Alpha's address, then fire your first transfer from the dashboard.

---

## Python CLI

A standalone Python script that mirrors the full transfer flow without the browser UI.

```bash
cd python/
pip install -r requirements.txt
cp ../env.example .env   # or copy your .env here

# Provision wallets
python agent_transfer.py provision

# Check balances
python agent_transfer.py balances

# Send $0.10 USDC
python agent_transfer.py transfer 0.10

# View history
python agent_transfer.py history

# Check a transaction
python agent_transfer.py status <transaction-id>
```

See [`python/README.md`](python/README.md) for full details.

---

## Smart Contracts

### DisputeResolver

An AI-powered USDC escrow dispute resolution contract deployed on Arc Testnet.

| Feature | Detail |
|---|---|
| Deployed address | See `AGENTS.md` |
| Stake model | Both parties stake equal USDC; winner takes the pot |
| Verdict | AI oracle submits verdict onchain; owner can override |
| Payouts | Pull-payment via `claimPayout(disputeId)` — O(1), blocklist-safe |
| Timeouts | 7-day join window; 14-day verdict window; refundable on expiry |

```bash
# Build
bun run contracts:build

# Test
bun run contracts:test
```

---

## API Routes

The backend (`server.ts`) exposes these routes (proxied through Vite at `/api/*`):

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/agents/provision` | Create wallet set + 2 SCA wallets |
| `GET` | `/api/agents` | List agents with live USDC balances |
| `POST` | `/api/transfer` | Initiate USDC transfer |
| `GET` | `/api/tx/:id` | Poll transaction state |
| `GET` | `/api/history` | In-memory transfer history |

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `CIRCLE_DEVELOPER_CONTROLLED_API_KEY` | Yes | Circle API key (`PREFIX:ID:SECRET`) |
| `CIRCLE_ENTITY_SECRET` | Yes | 32-byte hex entity secret |

> **Security:** Never commit `.env`. The `.gitignore` already excludes it.  
> Entity secrets are one-time registered per Circle entity — store the recovery file securely.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

MIT — see [LICENSE](LICENSE).
# Agent-to-Agent USDC Transfer

> Programmatic USDC transfers between AI agents on Arc Testnet. Full-stack demo using Circle developer-controlled wallets, a React dashboard, Bun backend, Python CLI, and a Solidity USDC escrow dispute resolver.

![Chain](https://img.shields.io/badge/chain-Arc%20Testnet-122d45?style=flat-square)
![Token](https://img.shields.io/badge/token-USDC-2775CA?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-8dd89f?style=flat-square)
![CI](https://img.shields.io/github/actions/workflow/status/your-username/agent-to-agent-usdc/ci.yml?style=flat-square&label=CI)

Two wallets (Agent Alpha and Agent Beta) are provisioned via the Circle SDK. A dashboard lets you fire transfers, watch real-time transaction state, and inspect the history — all onchain on Arc, where USDC is the native gas token.

---

## Features

- **Two agent wallets** provisioned in one click via Circle developer-controlled wallets SDK
- **Live USDC balances** polled from the Circle API
- **Programmatic transfers** with real-time state transitions (INITIATED → SENT → COMPLETE)
- **Arc Testnet explorer** links for every confirmed transaction
- **Python CLI** companion (`python/agent_transfer.py`) — run the same flow without a browser
- **Solidity contracts** (DisputeResolver) for USDC escrow-based dispute resolution

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite, TypeScript, Tailwind CSS, Framer Motion |
| Backend | Bun + Node HTTP server (TypeScript) |
| Onchain | wagmi v2, viem v2, ConnectKit |
| Wallets | Circle Developer-Controlled Wallets SDK |
| Contracts | Solidity 0.8.28, Foundry, OpenZeppelin 5.1.0 |
| Chain | Arc Testnet (Chain ID: 5042002) |
| Token | USDC (Arc Testnet: `0x3600000000000000000000000000000000000000`) |
| Python CLI | Python 3.11+, httpx, python-dotenv |

---

## Project Structure

```
.
├── src/                          # React frontend
│   ├── App.tsx                   # App root
│   ├── components/
│   │   ├── AgentTransferDashboard.tsx   # Main dashboard
│   │   └── DisputeDashboard.tsx         # Dispute resolver UI
│   ├── onchain-facts.ts          # Chain/USDC addresses (generated)
│   ├── onchain-money.ts          # USDC amount helpers
│   └── onchain-wait.ts           # Transaction state polling
├── server.ts                     # Bun backend — Circle SDK API routes
├── contracts/
│   ├── DisputeResolver.sol       # USDC escrow dispute contract
│   └── test/                     # Foundry tests
├── python/
│   ├── agent_transfer.py         # Python CLI companion
│   ├── requirements.txt
│   └── README.md
├── .github/
│   ├── workflows/ci.yml          # Type-check + lint on push/PR
│   └── ISSUE_TEMPLATE/           # Bug report + feature request templates
├── foundry.toml
└── package.json
```

---

## Quick Start

### Prerequisites

- [Bun](https://bun.sh) v1.0+
- [Foundry](https://getfoundry.sh) (for contract work)
- A [Circle developer account](https://console.circle.com) with an API key + entity secret

### 1. Install dependencies

```bash
bun install
```

### 2. Configure environment

```bash
cp env.example .env
```

Edit `.env`:

```
CIRCLE_DEVELOPER_CONTROLLED_API_KEY=your_api_key_here
CIRCLE_ENTITY_SECRET=your_entity_secret_here
```

### 3. Run the app

```bash
# Terminal 1 — backend
bun server.ts

# Terminal 2 — frontend
bun run dev
```

Open `http://localhost:5173` and click **Provision Agents**.

### 4. Fund Agent Alpha

Use the testnet faucet to send USDC to Agent Alpha's address, then fire your first transfer from the dashboard.

---

## Python CLI

A standalone Python script that mirrors the full transfer flow without the browser UI.

```bash
cd python/
pip install -r requirements.txt
cp ../env.example .env   # or copy your .env here

# Provision wallets
python agent_transfer.py provision

# Check balances
python agent_transfer.py balances

# Send $0.10 USDC
python agent_transfer.py transfer 0.10

# View history
python agent_transfer.py history

# Check a transaction
python agent_transfer.py status <transaction-id>
```

See [`python/README.md`](python/README.md) for full details.

---

## Smart Contracts

### DisputeResolver

An AI-powered USDC escrow dispute resolution contract deployed on Arc Testnet.

| Feature | Detail |
|---|---|
| Deployed address | See `AGENTS.md` |
| Stake model | Both parties stake equal USDC; winner takes the pot |
| Verdict | AI oracle submits verdict onchain; owner can override |
| Payouts | Pull-payment via `claimPayout(disputeId)` — O(1), blocklist-safe |
| Timeouts | 7-day join window; 14-day verdict window; refundable on expiry |

```bash
# Build
bun run contracts:build

# Test
bun run contracts:test
```

---

## API Routes

The backend (`server.ts`) exposes these routes (proxied through Vite at `/api/*`):

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/agents/provision` | Create wallet set + 2 SCA wallets |
| `GET` | `/api/agents` | List agents with live USDC balances |
| `POST` | `/api/transfer` | Initiate USDC transfer |
| `GET` | `/api/tx/:id` | Poll transaction state |
| `GET` | `/api/history` | In-memory transfer history |

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `CIRCLE_DEVELOPER_CONTROLLED_API_KEY` | Yes | Circle API key (`PREFIX:ID:SECRET`) |
| `CIRCLE_ENTITY_SECRET` | Yes | 32-byte hex entity secret |

> **Security:** Never commit `.env`. The `.gitignore` already excludes it.  
> Entity secrets are one-time registered per Circle entity — store the recovery file securely.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

MIT — see [LICENSE](LICENSE).
