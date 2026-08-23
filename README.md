# CredFi (Veriflow) — BOT Chain Builder Challenge #2 (RWA Track)

CredFi turns real-world receivables — invoices, royalty streams, rental
income — into instantly-funded, collateral-backed capital on-chain.
Originators tokenize a claim as an ERC-721, post partial stablecoin
collateral sized by an on-chain risk score, and get funded by investors
same-day. Repayment (principal + fixed yield) settles through a challenge
window; disputes are resolved by a 2-of-3 arbitrator multisig scoped to the
individual claim.

**🟢 Live on BOT Chain Mainnet — see [Mainnet Deployment](#mainnet-deployment-chain-id-677) below.**

## How it works

```
Originator                    VeriflowClaimNFT              VeriflowClaimVault
    │  mintClaim()                    │                              │
    ├─────────────────────────────────▶  ERC-721 + claim metadata    │
    │  lockCollateral()                (RiskOracle sets the ratio)   │
    ├─────────────────────────────────▶  collateral locked           │
    │                                  │                              │
Investor                                │      fundClaim()            │
    ├────────────────────────────────────────────────────────────────▶
    │                                  │   cash-now payout to originator,
    │                                  │   capped at claimAmount − requiredCollateral
    │                                  │                              │
Originator                              │   depositRepayment()        │
    ├────────────────────────────────────────────────────────────────▶
    │                                  │   principal + 10% yield      │
    │                                  │                              │
    │              challenge window elapses, no dispute               │
Investor  ◀─────────────────────────────────────────────  distributeToInvestors()
    │                                  │                              │
    │        dispute raised?  →  ArbitratorMultisig (2-of-3 vote)  →  slash / release
```

- **Funding is capped by risk**: `RiskOracle` sets a collateral ratio per
  claim type (Invoice 10%, Royalty 15%, Rental 20%); the vault refuses to
  fund more than `claimAmount − requiredCollateral` and reverts loudly if
  the oracle isn't configured, rather than silently skipping the check.
- **Payout is immediate**: `fundClaim` transfers stablecoin straight to the
  claim's originator in the same transaction — no separate withdrawal step.
- **Yield is fixed and enforced on-chain**: repayment must cover principal
  plus a 10% yield (`YIELD_BPS = 1000`) or `depositRepayment` reverts.
- **Disputes are claim-scoped**: `ArbitratorMultisig` tracks approvals/
  rejections per `claimId`, so a vote on one claim can't resolve another.

## Project Structure

```
BOTchain/
├── contracts/               # Solidity smart contracts
│   ├── RiskOracle.sol           # Per-claim-type collateral ratios
│   ├── ArbitratorMultisig.sol   # 2-of-3 claim-scoped dispute resolution
│   ├── VeriflowClaimVault.sol   # Funding, repayment, yield, distribution
│   ├── VeriflowClaimNFT.sol     # ERC-721 claim identity + collateral
│   └── test/MockStablecoin.sol  # 18-decimal mock USD for testing
├── scripts/
│   ├── deploy.ts                    # Full deploy + wiring + test fixtures
│   ├── verify-bytecode-markers.mjs  # Confirms a build contains expected fixes
│   ├── demo-flow.ts / demo-dispute-flow.ts
│   └── seed-active-positions.mjs
├── deployments/              # Per-chain deployment records (addresses + tx hashes)
├── test/                     # Hardhat/Mocha tests (TypeScript)
├── frontend/                 # React + TypeScript dApp (Vite, wagmi, viem)
├── hardhat.config.ts
├── tsconfig.json
└── .env.example
```

## Chain Configuration

| Network            | Chain ID | RPC                      | Explorer                   |
|---------------------|----------|---------------------------|------------------------------|
| BOT Chain Mainnet  | 677      | https://rpc.botchain.ai  | https://scan.botchain.ai   |
| BOT Chain Testnet  | 968      | https://rpc.bohr.life    | https://scan.bohr.life     |

Source: https://dev-docs.botchain.ai/docs/Developers/quick-guide/

---

## Mainnet Deployment (chain ID 677)

### Contracts

| Contract | Address | Explorer |
|---|---|---|
| MockStablecoin (mUSD) | `0x680A55c0Db4B44def9d88cCBF450C1f5dd37fd9a` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x680A55c0Db4B44def9d88cCBF450C1f5dd37fd9a#code) |
| RiskOracle | `0x87934d5E1A61be3Bb06FE54AC7e21E7704731d1C` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x87934d5E1A61be3Bb06FE54AC7e21E7704731d1C#code) |
| ArbitratorMultisig | `0xE33Dc788C060cb77F79A2AFF96Ca685f6C018721` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0xE33Dc788C060cb77F79A2AFF96Ca685f6C018721#code) |
| VeriflowClaimVault | `0x38703a57c5f8eB2F8d1576A3d2B4B35A10D66FA6` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x38703a57c5f8eB2F8d1576A3d2B4B35A10D66FA6#code) |
| VeriflowClaimNFT | `0x8a747D78811066F58A0dDe04E82165A02fA37725` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x8a747D78811066F58A0dDe04E82165A02fA37725#code) |

---

## Testnet Deployment (chain ID 968, reference)

| Contract | Address |
|---|---|
| MockStablecoin | `0x2DC17aD3AF5E195c96d53c916F9e3c95FEea4bA2` |
| RiskOracle | `0xA94B7Fd8ac55Fd481B559caC159309Bc779f34c0` |
| ArbitratorMultisig | `0x829A3Da07b4f043789496d77F0C304DE7aFc67E4` |
| VeriflowClaimVault | `0xAC0B43de7893Ec6CaBFd25940987779668F1204B` |
| VeriflowClaimNFT | `0xc4e7dD84165a5247637E106f4Bbe9520876D05b3` |

---

## Getting Started

```bash
cp .env.example .env
# fill in PRIVATE_KEY (must be one of ARBITRATOR_1/2/3 — see below),
# ARBITRATOR_1/2/3, and EXPECTED_CHAIN_ID

npx hardhat compile
npx hardhat test
```

### Deploying

`scripts/deploy.ts` deploys all five contracts in dependency order
(Stablecoin → Oracle → Multisig → Vault → NFT), wires every cross-contract
reference, and allowlists/mints test fixtures — all in one script, all
transactions logged to `deployments/<chainId>.json`.

```bash
# Testnet
EXPECTED_CHAIN_ID=968 npx hardhat run scripts/deploy.ts --network botchain_testnet

# Mainnet
EXPECTED_CHAIN_ID=677 npx hardhat run scripts/deploy.ts --network botchain
```

Guardrails baked into the script:
- Refuses to run unless `EXPECTED_CHAIN_ID` matches the connected chain.
- Refuses to run if the deployer has zero balance.
- Refuses to run unless the deployer is one of `ARBITRATOR_1/2/3` —
  `ArbitratorMultisig.setClaimNFT`/`setClaimVault` gate on multisig
  ownership (`isOwner[msg.sender]`), not `Ownable`, so a non-arbitrator
  deployer would revert mid-deploy.

Optional env vars: `TEST_ORIGINATORS` / `TEST_INVESTORS` (comma-separated
addresses to allowlist/fund beyond the deployer), `TEST_MINT_AMOUNT`
(default 1,000,000 mUSD), `CHALLENGE_WINDOW_SECONDS` (default 300).

### Verifying

```bash
npx hardhat verify blockscout --network botchain <address> [constructorArgs...]
# or, for constructor args that don't fit on the command line (arrays, etc.):
npx hardhat verify blockscout --network botchain --constructor-args-path <file>.cjs <address>
```

BOTScan is a Blockscout-backed explorer; the `blockscout` verify task works
without an API key. The `etherscan`-compatible path at the same explorer
does require one (`BOTCHAIN_EXPLORER_API_KEY`).

### Confirming a build matches the intended source before deploying

```bash
rm -rf artifacts cache && npx hardhat compile
node scripts/verify-bytecode-markers.mjs
```

Checks that the compiled artifact was built from the exact `.sol` files on
disk, and that each contract's runtime bytecode actually contains the
selectors/event topics for its intended behavior (funding cap enforcement,
fail-loud RiskOracle check, yield calculation, immediate payout, claim-scoped
dispute resolution) — with a negative control confirming no pre-fix code
paths remain.

### Testing

```bash
npx hardhat test
```

21 passing — unit coverage for `RiskOracle` and `VeriflowClaimNFT`, plus
integration coverage for the full fund → repay → challenge-window →
distribute happy path, the risk-oracle-not-set revert, the funding-cap
enforcement, and the dispute → arbitrator-slash path.

## Frontend

React + TypeScript (Vite), wagmi + viem for wallet/contract interaction.

```bash
cd frontend
npm install
npm run dev
```

`frontend/src/config/chains.ts` holds a single `activeChain` switch point;
`frontend/src/config/contracts.ts` resolves the address book for whichever
chain is active and throws at import time if that chain has no deployment
recorded, so the two can't silently drift apart.
