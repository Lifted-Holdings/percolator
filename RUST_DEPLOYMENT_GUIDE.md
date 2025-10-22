# Percolator: Rust Deployment Configuration and Usage Guide

**Version:** 1.0  
**Status:** Development → Staging → Production Roadmap  
**Last Updated:** October 22, 2025

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Development Environment Setup](#development-environment-setup)
5. [Building the Programs](#building-the-programs)
6. [Testing Strategy](#testing-strategy)
7. [Local Deployment](#local-deployment)
8. [Front-End Integration Guide](#front-end-integration-guide)
9. [Staging Deployment](#staging-deployment)
10. [Production Deployment](#production-deployment)
11. [Monitoring and Operations](#monitoring-and-operations)
12. [Security Considerations](#security-considerations)
13. [Troubleshooting](#troubleshooting)
14. [Production Roadmap](#production-roadmap)
15. [Appendix](#appendix)

---

## Executive Summary

Percolator is a **sharded perpetual exchange protocol** for Solana that provides unprecedented capital efficiency through cross-slab portfolio margin netting. This document provides comprehensive instructions for:

- Setting up a development environment
- Building and testing Rust-based Solana programs
- Integrating with front-end applications
- Deploying to testnet and production
- Operating and monitoring in production

**Key Value Propositions:**
- **Capital Efficiency**: Cross-margin across multiple trading venues (slabs)
- **Scalability**: 10 MB per-slab state budget with O(1) allocation
- **Security**: Capability-based authorization with time-limited scoped debits
- **Performance**: Zero-allocation after initialization, deterministic matching

---

## Architecture Overview

### Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Frontend Application                    │
│  (React/Vue/Angular + TypeScript SDK + Web3 Wallet)        │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ JSON-RPC / WebSocket
                 ▼
┌─────────────────────────────────────────────────────────────┐
│                    Solana Blockchain                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Router Program (Global Coordinator)          │  │
│  │  - Collateral Management (Vaults per asset)          │  │
│  │  - Portfolio Margin (Cross-slab netting)             │  │
│  │  - Capability System (Time-limited debits)           │  │
│  │  - Registry (Governance-controlled slab whitelist)   │  │
│  │  Program ID: RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr │
│  └────────────┬─────────────────────────────────────────┘  │
│               │                                              │
│               │ Cross-Program Invocation (CPI)              │
│               ▼                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Slab Program (LP-run Perp Engine)            │  │
│  │  - 10 MB State Budget (compile-time enforced)        │  │
│  │  - Order Book (Price-time priority matching)         │  │
│  │  - Position Management (VWAP tracking)               │  │
│  │  - Reserve-Commit Two-Phase Execution                │  │
│  │  Program ID: SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Data Flow (Typical Order Execution)

```
1. User initiates order via Frontend
2. Frontend signs transaction with wallet
3. Router receives order, reads QuoteCache from multiple slabs
4. Router splits order optimally across slabs
5. Router pledges collateral to escrow (per slab)
6. Router creates time-limited capabilities (Caps)
7. Router CPIs to each slab's commit_fill instruction
8. Slabs execute fills, write FillReceipts
9. Router aggregates receipts, updates portfolio
10. Router calculates net exposure across all slabs
11. Router checks margin requirements (IM/MM)
12. Transaction succeeds/fails atomically
13. Frontend updates UI with new positions
```

### v0 Simplified Architecture (Current)

The current v0 implementation focuses on proving the core capital efficiency thesis with minimal complexity:

- **Slab**: Single 4KB account with QuoteCache (best 4 bid/ask levels) + minimal book
- **Router**: Direct CPI coordination, portfolio netting, margin checks
- **~1,000 LOC** vs ~5,000 LOC for full v1 architecture

---

## Prerequisites

### System Requirements

**Hardware:**
- **CPU**: 4+ cores recommended (8+ for production builds)
- **RAM**: 8 GB minimum, 16 GB recommended
- **Storage**: 20 GB free space (for Solana toolchain + builds)
- **Network**: Stable internet connection for RPC access

**Operating Systems:**
- Linux (Ubuntu 20.04+, Debian 11+)
- macOS (11.0+)
- Windows (via WSL2)

### Required Software

1. **Rust Toolchain** (1.75.0+)
2. **Solana CLI** (1.18.0+)
3. **Node.js** (18.0+) for front-end integration
4. **Git** (2.30+)
5. **Docker** (optional, for containerized deployments)

### Recommended Development Tools

- **IDE**: VS Code with rust-analyzer extension
- **Database**: PostgreSQL 14+ (for indexer/analytics)
- **Monitoring**: Prometheus + Grafana
- **Version Control**: Git with GitHub/GitLab

---

## Development Environment Setup

### Step 1: Install Rust

```bash
# Install Rust via rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Source the environment
source $HOME/.cargo/env

# Verify installation
rustc --version  # Should be 1.75.0 or higher
cargo --version

# Add wasm32-unknown-unknown target (optional, for wasm builds)
rustup target add wasm32-unknown-unknown
```

### Step 2: Install Solana CLI Tools

```bash
# Install Solana toolchain
sh -c "$(curl -sSfL https://release.solana.com/v1.18.0/install)"

# Add to PATH (add to ~/.bashrc or ~/.zshrc for persistence)
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"

# Verify installation
solana --version  # Should be 1.18.0 or higher
solana-keygen --version

# Set cluster to localhost for development
solana config set --url localhost

# Generate a keypair for testing (if you don't have one)
solana-keygen new --outfile ~/.config/solana/id.json

# Check your address
solana address
```

### Step 3: Install Additional Solana Tools

```bash
# Install Anchor (optional, for testing frameworks)
cargo install --git https://github.com/coral-xyz/anchor anchor-cli --locked

# Install SPL Token CLI
cargo install spl-token-cli

# Install Solana Program Library CLI tools
cargo install cargo-build-sbf
```

### Step 4: Clone and Build Percolator

```bash
# Clone the repository
git clone https://github.com/Lifted-Holdings/percolator.git
cd percolator

# Build all workspace members
cargo build --release

# Expected output: builds router, slab, oracle, common libraries
# Build artifacts in: target/release/
```

### Step 5: Verify Build

```bash
# Run all unit tests
cargo test --release

# Expected: 53 tests passing
# - percolator-common: 27 tests
# - percolator-router: 7 tests
# - percolator-slab: 19 tests

# Run specific package tests
cargo test --package percolator-router --release
cargo test --package percolator-slab --release
cargo test --package percolator-common --release
```

### Step 6: Set Up Development Config

```bash
# Create local config directory
mkdir -p .local

# Create development configuration
cat > .local/dev-config.toml << EOF
[network]
cluster = "localhost"
rpc_url = "http://localhost:8899"
ws_url = "ws://localhost:8900"

[programs]
router_program_id = "RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr"
slab_program_id = "SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk"

[testing]
airdrop_amount = 10_000_000_000  # 10 SOL in lamports

[logging]
level = "debug"
EOF

# Add .local/ to .gitignore (should already be there)
echo ".local/" >> .gitignore
```

---

## Building the Programs

### Standard Library Builds (for testing)

```bash
# Build all programs in debug mode
cargo build

# Build in release mode (optimized)
cargo build --release

# Build specific program
cargo build --package percolator-router --release
cargo build --package percolator-slab --release

# Clean build artifacts
cargo clean
```

### Solana BPF Builds (for deployment)

```bash
# Build router program for Solana BPF
cargo build-sbf --manifest-path programs/router/Cargo.toml

# Build slab program for Solana BPF
cargo build-sbf --manifest-path programs/slab/Cargo.toml

# Build all BPF programs using convenience script
./build-all-bpf.sh

# Expected output:
# - target/deploy/percolator_router.so
# - target/deploy/percolator_slab.so
# - target/deploy/percolator_oracle.so (if implemented)
```

### Build Profiles

The project uses three build profiles:

1. **dev**: Fast compilation, no optimizations
   ```bash
   cargo build  # Uses dev profile by default
   ```

2. **release**: Optimized for testing
   ```bash
   cargo build --release
   ```

3. **sbf**: Optimized for Solana BPF deployment
   ```bash
   cargo build-sbf  # Uses sbf profile
   # Includes: panic = "abort", LTO, single codegen unit
   ```

### Verifying Build Artifacts

```bash
# Check BPF program sizes
ls -lh target/deploy/*.so

# Router and Slab should be < 500 KB each
# Larger sizes may indicate optimization issues

# Verify program IDs (optional)
solana-keygen pubkey target/deploy/percolator_router-keypair.json
solana-keygen pubkey target/deploy/percolator_slab-keypair.json
```

---

## Testing Strategy

### Unit Tests

Run component-level tests for individual modules:

```bash
# Run all unit tests
cargo test --lib

# Run with output
cargo test --lib -- --nocapture

# Run specific test
cargo test test_vwap_calculation

# Run tests in parallel (default)
cargo test --lib --release

# Run tests sequentially (if needed)
cargo test --lib -- --test-threads=1
```

**Test Coverage:**
- **percolator-common**: Math utilities, VWAP, PnL, margin calculations
- **percolator-router**: Vault, escrow, capability, portfolio operations
- **percolator-slab**: Pools, matching, reserves, order book management

### Integration Tests

Integration tests require a running Solana validator:

```bash
# Terminal 1: Start local validator
solana-test-validator \
  --reset \
  --ledger test-ledger \
  --limit-ledger-size 50000000

# Terminal 2: Run integration tests
cargo test --test integration -- --test-threads=1

# Or use the convenience script (when available)
./scripts/run-integration-tests.sh
```

### Property-Based Tests (when implemented)

```bash
# Run property tests with proptest
cargo test --features proptest

# Adjust iterations (default: 256)
PROPTEST_CASES=1000 cargo test --features proptest
```

### BPF Program Tests

```bash
# Deploy and test on local validator
solana-test-validator --reset &

# Wait for validator to start
sleep 5

# Deploy programs
solana program deploy target/deploy/percolator_router.so
solana program deploy target/deploy/percolator_slab.so

# Run e2e tests
./scripts/test-e2e.sh

# Stop validator
solana-test-validator --reset
```

---

## Local Deployment

### Step 1: Start Local Validator

```bash
# Start validator with mainnet state (for realistic testing)
solana-test-validator \
  --url https://api.mainnet-beta.solana.com \
  --clone <TOKEN_PROGRAM_ID> \
  --clone <ORACLE_PROGRAM_ID> \
  --reset \
  --ledger test-ledger

# Or start basic validator
solana-test-validator --reset --ledger test-ledger

# Verify validator is running
solana cluster-version
solana epoch-info
```

### Step 2: Airdrop SOL for Testing

```bash
# Airdrop to your keypair
solana airdrop 10

# Verify balance
solana balance

# Create additional test accounts
solana-keygen new --outfile ~/.config/solana/test-user-1.json
solana airdrop 10 ~/.config/solana/test-user-1.json
```

### Step 3: Deploy Programs

```bash
# Deploy router program
solana program deploy target/deploy/percolator_router.so

# Note the Program ID (should match RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr)
# If different, update program IDs in code and rebuild

# Deploy slab program
solana program deploy target/deploy/percolator_slab.so

# Verify deployments
solana program show RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr
solana program show SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk
```

### Step 4: Initialize Router State

```bash
# Create initialization transaction (pseudocode - implement in SDK)
# This would typically be done via a CLI tool or SDK

# Example structure (implement in TypeScript SDK):
# 1. Initialize registry
# 2. Create vault for USDC
# 3. Register first slab
# 4. Set initial risk parameters
```

### Step 5: Initialize Slab

```bash
# Initialize slab with instrument parameters
# Example for BTC-PERP:
# - Market ID: "BTC-PERP"
# - Contract size: 0.001 BTC (in base units)
# - Oracle: Pyth BTC/USD feed
# - Risk params: IM/MM rates

# This requires SDK implementation
```

### Step 6: Verify Deployment

```bash
# Check program accounts
solana account <ROUTER_PROGRAM_ID>
solana account <SLAB_PROGRAM_ID>

# Check vault accounts (when created)
solana account <VAULT_PDA>

# Monitor logs
solana logs RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr
```

---

## Front-End Integration Guide

### Overview

Front-end applications interact with Percolator through:
1. **Wallet Adapters** (Phantom, Solflare, etc.)
2. **TypeScript SDK** (to be implemented)
3. **Solana Web3.js**
4. **WebSocket subscriptions** for real-time updates

### TypeScript SDK Structure (Recommended)

```typescript
// percolator-sdk/
├── src/
│   ├── client/
│   │   ├── PercolatorClient.ts      // Main client class
│   │   ├── RouterClient.ts          // Router-specific methods
│   │   └── SlabClient.ts            // Slab-specific methods
│   ├── accounts/
│   │   ├── Portfolio.ts             // Portfolio account decoder
│   │   ├── Vault.ts                 // Vault account decoder
│   │   ├── Escrow.ts                // Escrow account decoder
│   │   ├── SlabState.ts             // Slab state decoder
│   │   └── types.ts                 // Account type definitions
│   ├── instructions/
│   │   ├── router.ts                // Router instruction builders
│   │   ├── slab.ts                  // Slab instruction builders
│   │   └── types.ts                 // Instruction type definitions
│   ├── utils/
│   │   ├── pda.ts                   // PDA derivation helpers
│   │   ├── math.ts                  // Fixed-point math utilities
│   │   └── errors.ts                // Error handling
│   └── index.ts                     // Main exports
├── tests/                           // SDK unit tests
├── examples/                        // Usage examples
└── package.json
```

### SDK Implementation Template

```typescript
// src/client/PercolatorClient.ts
import { Connection, PublicKey, Transaction, Keypair } from '@solana/web3.js';

export interface PercolatorConfig {
  connection: Connection;
  programIds: {
    router: PublicKey;
    slab: PublicKey;
  };
  wallet: Keypair;
}

export class PercolatorClient {
  constructor(private config: PercolatorConfig) {}

  // Portfolio operations
  async getPortfolio(user: PublicKey): Promise<Portfolio> {
    const portfolioPda = this.derivePortfolioPda(user);
    const accountInfo = await this.config.connection.getAccountInfo(portfolioPda);
    return decodePortfolio(accountInfo.data);
  }

  async createPortfolio(): Promise<string> {
    const instruction = createInitializePortfolioInstruction(
      this.config.wallet.publicKey,
      this.config.programIds.router
    );
    const transaction = new Transaction().add(instruction);
    return this.sendAndConfirmTransaction(transaction);
  }

  // Order operations
  async placeOrder(params: OrderParams): Promise<string> {
    // 1. Derive necessary PDAs
    // 2. Build execute_cross_slab instruction
    // 3. Sign and send transaction
    // 4. Return transaction signature
  }

  // Deposit/Withdraw
  async deposit(mint: PublicKey, amount: number): Promise<string> {
    const instruction = createDepositInstruction(
      this.config.wallet.publicKey,
      mint,
      amount,
      this.config.programIds.router
    );
    const transaction = new Transaction().add(instruction);
    return this.sendAndConfirmTransaction(transaction);
  }

  // PDA derivations
  private derivePortfolioPda(user: PublicKey): PublicKey {
    const [pda] = PublicKey.findProgramAddressSync(
      [Buffer.from("portfolio"), user.toBuffer()],
      this.config.programIds.router
    );
    return pda;
  }

  // Helper methods
  private async sendAndConfirmTransaction(tx: Transaction): Promise<string> {
    tx.recentBlockhash = (await this.config.connection.getLatestBlockhash()).blockhash;
    tx.feePayer = this.config.wallet.publicKey;
    tx.sign(this.config.wallet);
    return this.config.connection.sendRawTransaction(tx.serialize());
  }
}
```

### React Integration Example

```typescript
// src/App.tsx
import { useWallet } from '@solana/wallet-adapter-react';
import { Connection } from '@solana/web3.js';
import { PercolatorClient } from '@percolator/sdk';

function App() {
  const { publicKey, signTransaction } = useWallet();
  const connection = new Connection('https://api.mainnet-beta.solana.com');

  const client = new PercolatorClient({
    connection,
    programIds: {
      router: new PublicKey('RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr'),
      slab: new PublicKey('SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk'),
    },
    wallet: publicKey, // Adapt for wallet adapter
  });

  const handlePlaceOrder = async () => {
    try {
      const signature = await client.placeOrder({
        side: 'buy',
        quantity: 1.0,
        limitPrice: 50000,
        slabs: [new PublicKey('...')],
      });
      console.log('Order placed:', signature);
    } catch (error) {
      console.error('Failed to place order:', error);
    }
  };

  return (
    <div>
      <button onClick={handlePlaceOrder}>Place Order</button>
    </div>
  );
}
```

### WebSocket Subscription for Real-Time Updates

```typescript
// src/hooks/usePortfolioSubscription.ts
import { useEffect, useState } from 'react';
import { Connection, PublicKey } from '@solana/web3.js';

export function usePortfolioSubscription(
  connection: Connection,
  portfolioPda: PublicKey
) {
  const [portfolio, setPortfolio] = useState(null);

  useEffect(() => {
    const subscriptionId = connection.onAccountChange(
      portfolioPda,
      (accountInfo) => {
        const decoded = decodePortfolio(accountInfo.data);
        setPortfolio(decoded);
      },
      'confirmed'
    );

    return () => {
      connection.removeAccountChangeListener(subscriptionId);
    };
  }, [connection, portfolioPda]);

  return portfolio;
}
```

### API Endpoints for Front-End

**Recommended REST API Structure** (implement separately):

```
GET  /api/v1/markets           - List available markets
GET  /api/v1/market/:id        - Market details and stats
GET  /api/v1/orderbook/:id     - Order book snapshot
GET  /api/v1/trades/:id        - Recent trades
GET  /api/v1/portfolio/:user   - User portfolio
POST /api/v1/orders            - Place order (proxies to blockchain)
GET  /api/v1/orders/:user      - User open orders
GET  /api/v1/positions/:user   - User positions

# WebSocket endpoints
WS   /ws/orderbook/:id         - Order book updates
WS   /ws/trades/:id            - Trade stream
WS   /ws/portfolio/:user       - Portfolio updates
```

### Front-End Security Checklist

- [ ] **Never store private keys** in front-end code or localStorage
- [ ] Use **wallet adapters** for transaction signing
- [ ] Validate all **PDAs client-side** before submission
- [ ] Implement **rate limiting** on order submissions
- [ ] Use **HTTPS** for all API calls
- [ ] Implement **CSP headers** to prevent XSS
- [ ] Validate **amounts and prices** before submission
- [ ] Show **transaction confirmation** before signing
- [ ] Handle **wallet disconnection** gracefully

---

## Staging Deployment

### Environment: Solana Devnet

Devnet is a public testnet with airdrop capabilities, ideal for staging.

### Step 1: Configure for Devnet

```bash
# Set cluster to devnet
solana config set --url devnet

# Generate/use existing keypair
solana-keygen new --outfile ~/.config/solana/devnet-deployer.json

# Airdrop devnet SOL (5 SOL at a time, max 2 per hour)
solana airdrop 5 ~/.config/solana/devnet-deployer.json
solana airdrop 5 ~/.config/solana/devnet-deployer.json

# Check balance
solana balance
```

### Step 2: Build for Devnet

```bash
# Ensure latest code is built
git pull origin main
cargo build-sbf --manifest-path programs/router/Cargo.toml
cargo build-sbf --manifest-path programs/slab/Cargo.toml

# Verify builds
ls -lh target/deploy/*.so
```

### Step 3: Deploy to Devnet

```bash
# Deploy router
solana program deploy \
  --keypair ~/.config/solana/devnet-deployer.json \
  --url devnet \
  target/deploy/percolator_router.so

# Save Program ID (should match declared ID or update code)
ROUTER_PROGRAM_ID=<output_from_deploy>

# Deploy slab
solana program deploy \
  --keypair ~/.config/solana/devnet-deployer.json \
  --url devnet \
  target/deploy/percolator_slab.so

SLAB_PROGRAM_ID=<output_from_deploy>

# Verify deployments
solana program show $ROUTER_PROGRAM_ID --url devnet
solana program show $SLAB_PROGRAM_ID --url devnet
```

### Step 4: Initialize State on Devnet

```bash
# Create initialization scripts (TypeScript SDK)
# Run initialization sequence:
# 1. Initialize registry
# 2. Create vaults (USDC, SOL, etc.)
# 3. Register slabs
# 4. Set risk parameters
# 5. Initialize test slab with BTC-PERP market

# Example (implement in SDK):
node scripts/initialize-devnet.js
```

### Step 5: Configure Front-End for Devnet

```typescript
// config/devnet.ts
export const devnetConfig = {
  cluster: 'devnet',
  rpcUrl: 'https://api.devnet.solana.com',
  wsUrl: 'wss://api.devnet.solana.com',
  programIds: {
    router: 'RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr',
    slab: 'SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk',
  },
  markets: [
    { id: 'BTC-PERP', slab: '...' },
    { id: 'ETH-PERP', slab: '...' },
  ],
};
```

### Step 6: Staging Testing

**Test Scenarios:**

1. **User Onboarding Flow**
   - Connect wallet
   - Create portfolio
   - Deposit collateral
   - Verify balance

2. **Order Execution**
   - Place limit orders
   - Place market orders
   - Cancel orders
   - Verify fills

3. **Portfolio Management**
   - Open positions
   - Monitor P&L
   - Check margin requirements
   - Close positions

4. **Edge Cases**
   - Insufficient margin
   - Invalid prices
   - Oracle failures
   - Network congestion

5. **Load Testing**
   - Simulate 100+ concurrent users
   - Measure transaction throughput
   - Monitor error rates
   - Check RPC node stability

```bash
# Run load tests
npm run test:load -- --users 100 --duration 300s

# Monitor devnet performance
solana transaction-history --limit 100
```

---

## Production Deployment

### Environment: Solana Mainnet-Beta

**⚠️ WARNING**: Production deployment involves real assets. Complete all staging tests and security audits before proceeding.

### Pre-Production Checklist

- [ ] **Security Audit**: Complete formal audit by reputable firm
- [ ] **Code Freeze**: Freeze codebase, no changes during deployment
- [ ] **Staging Tests**: All tests pass on devnet
- [ ] **Load Tests**: System handles expected production load
- [ ] **Disaster Recovery Plan**: Documented and tested
- [ ] **Monitoring Setup**: Prometheus, Grafana, alerts configured
- [ ] **Legal Review**: Compliance with regulations
- [ ] **Insurance**: Protocol insurance arranged
- [ ] **Multisig**: Set up multisig for program upgrades
- [ ] **Bug Bounty**: Launch bug bounty program

### Step 1: Prepare Mainnet Deployment Keys

```bash
# Generate mainnet deployer keypair (SECURE THIS!)
solana-keygen new --outfile ~/.config/solana/mainnet-deployer.json

# Transfer SOL to deployer (need ~10 SOL for deployment + rent)
# DO NOT airdrop on mainnet - purchase and transfer

# Verify balance
solana balance --url mainnet-beta --keypair ~/.config/solana/mainnet-deployer.json

# Create upgrade authority multisig (e.g., 3-of-5)
# Use Squads Protocol or similar for multisig
```

### Step 2: Deploy to Mainnet

```bash
# FINAL CHECK: Ensure correct code version
git log -1
git status  # Should be clean

# Build final release
cargo build-sbf --manifest-path programs/router/Cargo.toml
cargo build-sbf --manifest-path programs/slab/Cargo.toml

# Verify build hashes (optional, for transparency)
sha256sum target/deploy/percolator_router.so
sha256sum target/deploy/percolator_slab.so

# Deploy router
solana program deploy \
  --keypair ~/.config/solana/mainnet-deployer.json \
  --url mainnet-beta \
  --upgrade-authority <MULTISIG_ADDRESS> \
  target/deploy/percolator_router.so

# Record Program ID
ROUTER_PROGRAM_ID=<deployed_program_id>

# Deploy slab
solana program deploy \
  --keypair ~/.config/solana/mainnet-deployer.json \
  --url mainnet-beta \
  --upgrade-authority <MULTISIG_ADDRESS> \
  target/deploy/percolator_slab.so

SLAB_PROGRAM_ID=<deployed_program_id>

# Verify deployments
solana program show $ROUTER_PROGRAM_ID --url mainnet-beta
solana program show $SLAB_PROGRAM_ID --url mainnet-beta
```

### Step 3: Initialize Production State

```bash
# Initialize with multisig authorization
# Use Squads or similar for transaction signing

# 1. Initialize registry (multisig required)
# 2. Create vaults for production assets (USDC, USDT, SOL)
# 3. Set conservative risk parameters:
#    - Higher IM/MM rates initially
#    - Lower position limits
#    - Shorter capability TTLs
# 4. Register first production slab
# 5. Initialize oracle feeds (Pyth, Switchboard)

# Example initialization (via multisig SDK):
node scripts/initialize-mainnet.js --multisig <SQUADS_ADDRESS>
```

### Step 4: Gradual Rollout Strategy

**Phase 1: Limited Beta (Weeks 1-2)**
- Whitelist 10-20 trusted users
- Single market (BTC-PERP)
- Low position limits ($10k max)
- 24/7 monitoring

**Phase 2: Public Beta (Weeks 3-4)**
- Open to public with limits
- Add ETH-PERP market
- Increase position limits to $50k
- Publish dashboards

**Phase 3: General Availability (Week 5+)**
- Remove whitelists
- Add more markets (SOL-PERP, etc.)
- Increase limits to $500k
- Enable cross-slab routing

**Phase 4: Full Production (Month 2+)**
- Multi-slab coordination
- Institutional clients
- API partners
- Unlimited (within risk params)

### Step 5: Production Configuration

```typescript
// config/mainnet.ts
export const mainnetConfig = {
  cluster: 'mainnet-beta',
  rpcUrl: 'https://api.mainnet-beta.solana.com',  // Or paid RPC (Triton, Helius)
  wsUrl: 'wss://api.mainnet-beta.solana.com',
  programIds: {
    router: 'RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr',
    slab: 'SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk',
  },
  vaults: {
    usdc: '<USDC_VAULT_PDA>',
    usdt: '<USDT_VAULT_PDA>',
    sol: '<SOL_VAULT_PDA>',
  },
  oracles: {
    pyth: {
      btcUsd: 'GVXRSBjFk6e6J3NbVPXohDJetcTjaeeuykUpbQF8UoMU',
      ethUsd: 'JBu1AL4obBcCMqKBBxhpWCNUt136ijcuMZLFvTP7iWdB',
      solUsd: 'H6ARHf6YXhGYeQfUzQNGk6rDNnLBQKrenN712K4AQJEG',
    },
  },
  riskParams: {
    initialMarginRate: 500,      // 5% (500 bps)
    maintenanceMarginRate: 300,  // 3% (300 bps)
    maxLeverage: 20,             // 20x
    capabilityTtl: 120,          // 2 minutes
  },
};
```

### Step 6: Post-Deployment Verification

```bash
# Verify all accounts created
solana account $ROUTER_PROGRAM_ID --url mainnet-beta
solana account $SLAB_PROGRAM_ID --url mainnet-beta
solana account $USDC_VAULT_PDA --url mainnet-beta

# Test basic operations with small amounts
# 1. Deposit 10 USDC
# 2. Place small order (0.001 BTC)
# 3. Cancel order
# 4. Withdraw

# Monitor initial transactions
solana logs $ROUTER_PROGRAM_ID --url mainnet-beta
```

---

## Monitoring and Operations

### Key Metrics to Monitor

**Program Health:**
- Transaction success rate (target: >99.9%)
- Average transaction time (target: <2 seconds)
- Failed transaction reasons
- Compute unit (CU) usage per instruction

**Business Metrics:**
- Total value locked (TVL)
- Daily active users (DAU)
- Trading volume (24h)
- Open positions count
- Liquidations count

**System Metrics:**
- RPC node response time
- WebSocket connection stability
- API endpoint latency
- Database query performance

**Risk Metrics:**
- Aggregate open interest
- Net delta per market
- Underwater positions (equity < MM)
- Largest positions (concentration risk)

### Prometheus Metrics

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'percolator'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'solana_node'
    static_configs:
      - targets: ['localhost:8899']

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:5432']
```

### Grafana Dashboards

**Dashboard 1: System Overview**
- Transaction volume (TPS)
- Error rates
- P50/P95/P99 latencies
- Active users

**Dashboard 2: Trading Activity**
- Order flow
- Fill rates
- Price charts
- Volume by market

**Dashboard 3: Risk Monitor**
- TVL trend
- Open interest by market
- Margin utilization
- Liquidations

**Dashboard 4: Infrastructure**
- RPC health
- Database connections
- API response times
- Disk/memory usage

### Alerting Rules

```yaml
# alerts.yml
groups:
  - name: percolator_critical
    rules:
      - alert: HighTransactionFailureRate
        expr: rate(failed_transactions[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Transaction failure rate above 1%"

      - alert: LowLiquidity
        expr: available_liquidity < 100000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Available liquidity below $100k"

      - alert: RPCNodeDown
        expr: up{job="solana_node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RPC node is down"

      - alert: UnderwaterPositions
        expr: count(equity < maintenance_margin) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "More than 10 positions underwater"
```

### Logging Strategy

**Log Levels:**
- **ERROR**: Critical issues requiring immediate action
- **WARN**: Potential issues to investigate
- **INFO**: Normal operations (deposits, withdraws, orders)
- **DEBUG**: Detailed execution traces (dev/staging only)

**Structured Logging Example:**

```rust
// In Rust program (use pinocchio-log)
log!("Order placed: user={}, market={}, side={}, qty={}, px={}",
     user_pubkey, market_id, side, quantity, price);

log!("WARN: Low liquidity: market={}, available={}", market_id, available);

log!("ERROR: Capability expired: user={}, cap_id={}", user_pubkey, cap_id);
```

**Log Aggregation:**
- Use **Loki** or **Elasticsearch** for log aggregation
- Retain logs for 90 days (compliance)
- Set up log-based alerts

### Incident Response

**Severity Levels:**

| Level | Response Time | Examples |
|-------|---------------|----------|
| P0 (Critical) | 15 minutes | Program halted, funds at risk |
| P1 (High) | 1 hour | Transaction failures, high latency |
| P2 (Medium) | 4 hours | API issues, degraded performance |
| P3 (Low) | 1 business day | Minor bugs, feature requests |

**Incident Playbook:**

1. **Detection**: Alert fires or user report
2. **Triage**: Assess severity, assign owner
3. **Investigation**: Check logs, metrics, recent changes
4. **Mitigation**: Apply hotfix or rollback
5. **Resolution**: Verify fix deployed
6. **Post-Mortem**: Document root cause, prevention measures

### Operational Runbooks

**Runbook 1: Upgrade Programs**
```bash
# 1. Build new version
cargo build-sbf

# 2. Test on devnet
solana program deploy --url devnet target/deploy/percolator_router.so

# 3. Get multisig approval
# (via Squads or similar)

# 4. Upgrade on mainnet (multisig)
solana program upgrade \
  --upgrade-authority <MULTISIG> \
  --url mainnet-beta \
  <PROGRAM_ID> \
  target/deploy/percolator_router.so

# 5. Verify upgrade
solana program show <PROGRAM_ID> --url mainnet-beta

# 6. Monitor for issues
watch -n 5 'solana logs <PROGRAM_ID> --url mainnet-beta | tail -20'
```

**Runbook 2: Emergency Program Halt**
```bash
# If critical bug detected, use circuit breaker (if implemented)
# Or transfer upgrade authority to revoke program (EXTREME)

# Preferred: Implement pausable flag in program logic
# Update registry to mark program as paused

# Notify users via:
# - Discord announcement
# - Twitter post
# - Front-end banner
```

**Runbook 3: Liquidation Processing**
```bash
# Monitor for underwater positions
./scripts/check-liquidations.sh

# If liquidations needed, liquidator bots should handle automatically
# Manual intervention only if bots fail

# Check liquidation queue
solana account <LIQUIDATION_QUEUE_PDA>

# If stuck, trigger manual liquidation
node scripts/manual-liquidate.js --position <POSITION_ID>
```

---

## Security Considerations

### Smart Contract Security

**Implemented Safeguards:**
1. **Capability System**: Time-limited, scoped debits prevent unauthorized fund movement
2. **PDA Verification**: All PDAs validated against expected seeds
3. **No Reentrancy**: Programs follow checks-effects-interactions pattern
4. **Overflow Protection**: All arithmetic uses checked operations
5. **State Isolation**: Slabs cannot access Router state directly

**Additional Recommendations:**
- [ ] **Formal Verification**: Use tools like Certora or Halmos
- [ ] **Fuzzing**: Continuous fuzzing with Honggfuzz or AFL
- [ ] **Audit**: Get 2+ audits from reputable firms (Zellic, OtterSec, Neodyme)
- [ ] **Bug Bounty**: Launch on ImmuneFi with $100k+ rewards
- [ ] **Upgrade Timelock**: Add 48h timelock on program upgrades
- [ ] **Emergency Pause**: Implement circuit breaker for critical bugs

### Operational Security

**Key Management:**
- Use **hardware wallets** (Ledger) for deployer keys
- Store **multisig keys** in separate secure locations
- Implement **MPC** for institutional clients
- Rotate **API keys** quarterly
- Use **HSMs** for production signing

**Access Control:**
- Implement **RBAC** for team members
- Require **MFA** for all production access
- Audit **access logs** monthly
- Separate **dev/staging/prod** credentials
- Use **bastion hosts** for server access

**Infrastructure:**
- Use **dedicated RPC nodes** (not public endpoints)
- Implement **rate limiting** on APIs
- Enable **DDoS protection** (Cloudflare)
- Encrypt **data at rest** (databases)
- Use **VPN** for internal services

### Economic Security

**Oracle Manipulation:**
- Use **multiple oracle sources** (Pyth + Switchboard)
- Implement **price deviation checks**
- Add **circuit breakers** for extreme price moves
- Monitor **oracle update frequency**

**MEV Protection:**
- Use **private mempools** (Jito) for sensitive transactions
- Implement **batch auctions** for fair ordering
- Add **slippage protection** on client side
- Monitor for **sandwich attacks**

**Liquidation Cascades:**
- Implement **staggered liquidations** (not all-at-once)
- Maintain **insurance fund** per slab
- Use **Dutch auction** for liquidation pricing
- Set **conservative margin requirements**

### Compliance and Legal

**Regulatory Considerations:**
- **KYC/AML**: May be required depending on jurisdiction
- **Accredited Investor**: Limits for US users
- **Derivatives Regulation**: CFTC/SEC compliance in US
- **GDPR**: Data privacy for EU users
- **Tax Reporting**: 1099 forms for US traders

**Disclaimers:**
- Display **terms of service** before trading
- Require **risk acknowledgment** for first-time users
- Show **no-guarantee** warnings
- Provide **volatility warnings** for leveraged products

---

## Troubleshooting

### Common Issues and Solutions

#### Build Issues

**Issue: `cargo build-sbf` not found**
```bash
# Solution: Install cargo-build-sbf
cargo install cargo-build-sbf

# Or install full Solana toolchain
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

**Issue: Slab size exceeds 10 MB**
```bash
# Solution: Check slab size at compile time
# The program should fail to compile if > 10 MB
# Reduce pool sizes in slab/src/lib.rs if needed

# Verify deployed program size
solana program show <SLAB_PROGRAM_ID> | grep "Data Length"
```

**Issue: Linker errors**
```bash
# Solution: Update Rust and Solana toolchain
rustup update stable
solana-install update

# Clean and rebuild
cargo clean
cargo build-sbf
```

#### Runtime Issues

**Issue: Transaction fails with "Insufficient funds"**
```bash
# Check user balance
solana balance

# Check rent requirements for new accounts
solana rent <ACCOUNT_SIZE>

# Solution: Ensure sufficient SOL for transaction fees + rent
```

**Issue: "Custom program error: 0x1" (Invalid instruction)**
```bash
# Check instruction data format
# Verify discriminator matches expected instruction

# Enable program logs
solana logs <PROGRAM_ID> --url <CLUSTER>

# Common causes:
# - Wrong discriminator byte
# - Mismatched account order
# - Invalid PDA derivation
```

**Issue: "Error: Account not found"**
```bash
# Check if account was created
solana account <ACCOUNT_PUBKEY>

# Verify PDA derivation
# Ensure initialization transaction succeeded

# Check for typos in program IDs or seeds
```

#### Integration Issues

**Issue: WebSocket disconnections**
```typescript
// Solution: Implement reconnection logic
const ws = new WebSocket(WS_URL);

ws.on('close', () => {
  console.log('WebSocket closed, reconnecting...');
  setTimeout(() => connectWebSocket(), 5000);
});

ws.on('error', (error) => {
  console.error('WebSocket error:', error);
});
```

**Issue: RPC rate limiting**
```bash
# Solution: Use paid RPC providers
# - Helius (helius.xyz)
# - Triton (triton.one)
# - QuickNode (quicknode.com)

# Or run your own validator
solana-validator --rpc-port 8899
```

**Issue: Transaction expired (blockhash expired)**
```typescript
// Solution: Use recent blockhash and retry logic
const { blockhash, lastValidBlockHeight } = 
  await connection.getLatestBlockhash('confirmed');

transaction.recentBlockhash = blockhash;
transaction.lastValidBlockHeight = lastValidBlockHeight;

// Send with confirmation
const signature = await connection.sendRawTransaction(
  transaction.serialize(),
  { skipPreflight: false, maxRetries: 3 }
);

await connection.confirmTransaction({
  signature,
  blockhash,
  lastValidBlockHeight,
});
```

### Debug Mode

Enable verbose logging for troubleshooting:

```bash
# Rust program debug logs (in development)
# Add to programs/router/src/lib.rs:
use pinocchio_log::log;
log!("Debug: instruction={:?}, accounts={:?}", ix_data, accounts);

# Solana CLI verbose output
solana --verbose program deploy ...
solana --verbose transaction send ...

# Monitor program logs in real-time
solana logs <PROGRAM_ID> --url <CLUSTER>

# Get transaction details
solana confirm <SIGNATURE> --url <CLUSTER> --verbose
```

---

## Production Roadmap

### Phase 1: Core Infrastructure (Current - Month 1)

**Milestone: v0 Deployment**
- [x] Router program (vault, portfolio, registry)
- [x] Slab program (order book, matching, positions)
- [x] Common library (math, types, utils)
- [x] Unit tests (53 tests passing)
- [ ] Integration tests (Surfpool or solana-program-test)
- [ ] TypeScript SDK (client library)
- [ ] CLI tools (for LPs and admins)

**Deliverables:**
- Deployable programs on devnet
- Basic SDK for front-end integration
- Documentation (this guide)
- Initial audit (informal)

### Phase 2: Enhanced Features (Months 2-3)

**Milestone: v1 Feature Complete**
- [ ] Multi-slab coordination (atomic cross-slab routing)
- [ ] Reserve-commit flow (two-phase execution)
- [ ] Capability system (time-limited debits)
- [ ] Funding rate mechanism (time-weighted calculations)
- [ ] Anti-toxicity controls (JIT penalty, ARG, kill bands)
- [ ] Liquidation engine (autonomous liquidations)
- [ ] Insurance pools (per-slab insurance)

**Deliverables:**
- Feature-complete v1 programs
- Comprehensive test suite (unit + integration + property)
- Advanced SDK features
- Front-end reference implementation

### Phase 3: Testing and Audit (Month 4)

**Milestone: Audit-Ready**
- [ ] Property-based tests (invariant checking)
- [ ] Fuzz testing (100M+ inputs)
- [ ] Load testing (1000+ TPS sustained)
- [ ] Chaos testing (network failures, oracle outages)
- [ ] Economic simulations (liquidation cascades)
- [ ] Formal verification (critical paths)
- [ ] External security audit (2+ firms)

**Deliverables:**
- Audit reports
- Test coverage >90%
- Performance benchmarks
- Security assessment

### Phase 4: Mainnet Beta (Month 5)

**Milestone: Limited Mainnet Launch**
- [ ] Deploy to mainnet with limits
- [ ] Whitelist 20-50 beta users
- [ ] Single market (BTC-PERP)
- [ ] Position limits ($10k max)
- [ ] 24/7 monitoring and support
- [ ] Bug bounty program ($100k+)

**Deliverables:**
- Mainnet deployment
- Operational runbooks
- Monitoring dashboards
- Incident response plan

### Phase 5: Public Launch (Month 6)

**Milestone: General Availability**
- [ ] Remove whitelist
- [ ] Add more markets (ETH, SOL, etc.)
- [ ] Increase position limits
- [ ] Enable cross-slab routing
- [ ] API partnerships
- [ ] Marketing campaign

**Deliverables:**
- Public mainnet launch
- Marketing materials
- User documentation
- Partner integrations

### Phase 6: Scaling and Growth (Months 7-12)

**Milestone: Production Scale**
- [ ] 10+ active slabs
- [ ] $100M+ TVL
- [ ] 1000+ daily active users
- [ ] Institutional onboarding
- [ ] Advanced order types
- [ ] Mobile app
- [ ] Governance token launch

**Deliverables:**
- Scaled infrastructure
- Mobile SDK
- Governance framework
- Institutional features

### Phase 7: Advanced Features (Year 2+)

**Long-Term Roadmap:**
- Multi-asset collateral (beyond USDC)
- Cross-chain bridges (Ethereum, Arbitrum)
- Options and exotic derivatives
- Lending and borrowing
- Automated market making (AMM integration)
- DAO governance
- L2 scaling solutions

---

## Appendix

### A. Program IDs

| Program | ID | Description |
|---------|-----|-------------|
| Router | `RoutR1VdCpHqj89WEMJhb6TkGT9cPfr1rVjhM3e2YQr` | Global coordinator |
| Slab | `SLabZ6PsDLh2X6HzEoqxFDMqCVcJXDKCNEYuPzUvGPk` | LP perp engine |

### B. PDA Seeds

**Router PDAs:**
```rust
// Vault: [b"vault", mint]
// Escrow: [b"escrow", user, slab, mint]
// Capability: [b"cap", user, slab, mint, nonce_u64]
// Portfolio: [b"portfolio", user]
// Registry: [b"registry"]
```

**Slab PDAs:**
```rust
// Slab State: [b"slab", market_id]
// Authority: [b"authority", slab]
```

### C. Error Codes

| Code | Name | Description |
|------|------|-------------|
| 0x01 | InvalidInstruction | Unknown instruction discriminator |
| 0x02 | InvalidAccount | Account validation failed |
| 0x03 | InsufficientFunds | Not enough balance |
| 0x04 | InsufficientMargin | Margin requirement not met |
| 0x05 | CapabilityExpired | Cap past expiry timestamp |
| 0x06 | CapabilityExhausted | Cap remaining = 0 |
| 0x07 | InvalidPrice | Price out of bounds |
| 0x08 | InvalidQuantity | Quantity out of bounds |
| 0x09 | OrderNotFound | Order ID not in book |
| 0x0A | PositionNotFound | User has no position |
| 0x0B | SlabNotRegistered | Slab not in registry |
| 0x0C | OracleStale | Oracle price too old |

### D. Useful Commands

```bash
# Generate new keypair
solana-keygen new --outfile ~/.config/solana/my-keypair.json

# Check balance
solana balance

# Airdrop SOL (devnet/testnet only)
solana airdrop 5

# Get program account data
solana account <PUBKEY>

# Get transaction details
solana confirm <SIGNATURE>

# Monitor program logs
solana logs <PROGRAM_ID>

# Get current slot
solana slot

# Get current epoch
solana epoch-info

# Transfer SOL
solana transfer <RECIPIENT> <AMOUNT>

# Create token account
spl-token create-account <MINT>

# Get token balance
spl-token balance <MINT>

# Transfer tokens
spl-token transfer <MINT> <AMOUNT> <RECIPIENT>
```

### E. Resources

**Documentation:**
- [Solana Docs](https://docs.solana.com/)
- [Pinocchio Docs](https://docs.rs/pinocchio/)
- [Solana Cookbook](https://solanacookbook.com/)
- [Anchor Book](https://book.anchor-lang.com/)

**Tools:**
- [Solana Explorer](https://explorer.solana.com/)
- [SolScan](https://solscan.io/)
- [Solana Beach](https://solanabeach.io/)
- [Step Finance](https://app.step.finance/)

**Community:**
- [Solana Discord](https://discord.gg/solana)
- [Solana Stack Exchange](https://solana.stackexchange.com/)
- [Percolator Discord](https://discord.gg/percolator) (TBD)

**Security:**
- [Zellic](https://zellic.io/)
- [OtterSec](https://osec.io/)
- [Neodyme](https://neodyme.io/)
- [ImmuneFi](https://immunefi.com/)

### F. Glossary

- **BPF**: Berkeley Packet Filter (Solana's VM)
- **CPI**: Cross-Program Invocation (calling another program)
- **PDA**: Program Derived Address (deterministic address)
- **SPL**: Solana Program Library (token standard)
- **IM**: Initial Margin (for opening positions)
- **MM**: Maintenance Margin (for keeping positions open)
- **VWAP**: Volume-Weighted Average Price
- **Slab**: LP-run perpetual exchange instance
- **Router**: Global coordinator and collateral custodian
- **Cap**: Capability (time-limited debit authorization)

### G. Contact and Support

**Development Team:**
- Email: dev@percolator.finance (TBD)
- Discord: discord.gg/percolator (TBD)
- Telegram: t.me/percolator (TBD)

**Security:**
- Security Email: security@percolator.finance (TBD)
- Bug Bounty: immunefi.com/percolator (TBD)

**Operations:**
- Status Page: status.percolator.finance (TBD)
- Monitoring: metrics.percolator.finance (TBD)

---

## Conclusion

This guide provides a comprehensive roadmap for deploying and operating Percolator in production. Follow the phases sequentially, complete all checklists, and prioritize security at every stage.

**Key Takeaways:**
1. **Start Small**: Test thoroughly on devnet before mainnet
2. **Gradual Rollout**: Whitelist → Public Beta → Full Production
3. **Security First**: Audits, bug bounties, multisig, monitoring
4. **Monitor Everything**: Logs, metrics, alerts, dashboards
5. **Plan for Failure**: Incident response, disaster recovery, backups

**Next Steps:**
1. Complete Phase 1 (Core Infrastructure)
2. Implement TypeScript SDK
3. Deploy to devnet
4. Build front-end integration
5. Conduct security audit
6. Launch mainnet beta

Good luck! 🚀

---

**Document Version**: 1.0  
**Last Updated**: October 22, 2025  
**Maintainers**: Percolator Core Team  
**License**: Apache-2.0
