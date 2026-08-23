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

Deployed 2026-08-22. Full machine-readable record: [`deployments/677.json`](deployments/677.json).
Deployer / `ARBITRATOR_1`: `0x68343Aa0598b7FCAA102769D172e59cdDfae10f2`.

### Contracts

| Contract | Address | Explorer | Verified |
|---|---|---|---|
| MockStablecoin (mUSD) | `0x680A55c0Db4B44def9d88cCBF450C1f5dd37fd9a` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x680A55c0Db4B44def9d88cCBF450C1f5dd37fd9a#code) | ✅ |
| RiskOracle | `0x87934d5E1A61be3Bb06FE54AC7e21E7704731d1C` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x87934d5E1A61be3Bb06FE54AC7e21E7704731d1C#code) | ✅ |
| ArbitratorMultisig | `0xE33Dc788C060cb77F79A2AFF96Ca685f6C018721` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0xE33Dc788C060cb77F79A2AFF96Ca685f6C018721#code) | ✅ |
| VeriflowClaimVault | `0x38703a57c5f8eB2F8d1576A3d2B4B35A10D66FA6` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x38703a57c5f8eB2F8d1576A3d2B4B35A10D66FA6#code) | ✅ |
| VeriflowClaimNFT | `0x8a747D78811066F58A0dDe04E82165A02fA37725` | [scan.botchain.ai ↗](https://scan.botchain.ai/address/0x8a747D78811066F58A0dDe04E82165A02fA37725#code) | ⏳ pending (explorer rate-limited during submission — retry with the command below) |

To retry NFT verification:
```bash
npx hardhat verify blockscout --network botchain \
  --constructor-args-path scripts/verify-args-nft.cjs \
  0x8a747D78811066F58A0dDe04E82165A02fA37725
```

### Configuration at deploy time

| Parameter | Value |
|---|---|
| Challenge window | 120 seconds |
| Fixed yield (`YIELD_BPS`) | 1000 bps (10%) |
| Collateral ratio — Invoice | 1000 bps (10%) |
| Collateral ratio — Royalty | 1500 bps (15%) |
| Collateral ratio — Rental | 2000 bps (20%) |
| Arbitrator quorum | 2-of-3 |
| `ARBITRATOR_1` | `0x68343Aa0598b7FCAA102769D172e59cdDfae10f2` |
| `ARBITRATOR_2` | `0x150C659d09fc5B9d53Cf2F18234517E8C702a614` |
| `ARBITRATOR_3` | `0x48D8f72D20D8F9C0b7abab30AB846f4D82a08dc7` |

### Deployment transactions

| Step | Tx hash | Block |
|---|---|---|
| Deploy MockStablecoin | [`0x6d6fc5c2…5c931`](https://scan.botchain.ai/tx/0x6d6fc5c254629eab04d0155c4912a2b5989dc811f11b659e89d12c9921a5c931) | 20566286 |
| Deploy RiskOracle | [`0x5d0caac7…0fe61`](https://scan.botchain.ai/tx/0x5d0caac7a1eedc97fa4a492baf10bea04e4927c6022c214f1af69f13b000fe61) | 20566289 |
| Deploy ArbitratorMultisig | [`0xc44d259f…da7048e`](https://scan.botchain.ai/tx/0xc44d259fe1874087c97f26badeb5224b5f3d5a89db9f50f133153a10cda7048e) | 20566292 |
| Deploy VeriflowClaimVault | [`0xbc6bb465…10cad4da8`](https://scan.botchain.ai/tx/0xbc6bb4650547eab4634ca17c37bbd570c8ec077e0bae990399d814a10cad4da8) | 20566297 |
| Deploy VeriflowClaimNFT | [`0xe3a0c326…90c674a8`](https://scan.botchain.ai/tx/0xe3a0c326eabd8f8b37aef1da6fdd7fb1a3d8ded17a9a2afa1d7247fa90c674a8) | 20566303 |
| `nft.setArbitrator` | [`0xff1e1374…5e788df08`](https://scan.botchain.ai/tx/0xff1e13743ee0c81a7b9325f8227256d01a3b0eb18633b252faf1ae45e788df08) | 20566308 |
| `nft.setVault` | [`0x6c30dbdc…d2a5717fbd7fd`](https://scan.botchain.ai/tx/0x6c30dbdc42a12d18243a036ce29182a7413b1db05b567ad1aa4d2a5717fbd7fd) | 20566310 |
| `vault.setArbitrator` | [`0x35014aa9…5f3df6f435e0e3`](https://scan.botchain.ai/tx/0x35014aa993e87d9fdad9554784f0a0e0dc4d5d44dc79c597de5f3df6f435e0e3) | 20566313 |
| `vault.setClaimNFT` | [`0x3a9879ec…5e95f403526ee`](https://scan.botchain.ai/tx/0x3a9879ec7b6adb60922b29198763d52f141461499815b62cf5b5e95f403526ee) | 20566316 |
| `vault.setRiskOracle` | [`0x0f93f23c…c5ab0df4f770f4e8460`](https://scan.botchain.ai/tx/0x0f93f23c15b9815379398ece948985ceea9148e692e06c5ab0df4f770f4e8460) | 20566319 |
| `multisig.setClaimNFT` | [`0x42110151…a007dfbd3a20343e78e9`](https://scan.botchain.ai/tx/0x421101511b29bef09d69807383bec937e378ac52c0d4a007dfbd3a20343e78e9) | 20566321 |
| `multisig.setClaimVault` | [`0x19074528…f32696590ee989fdc3`](https://scan.botchain.ai/tx/0x1907452835deb15070ef064ede5030e83e31dabf8eb6e1f32696590ee989fdc3) | 20566323 |
| Allowlist `0x68343Aa0…` (deployer) | [`0x36f06271…375911109a9ba9063154`](https://scan.botchain.ai/tx/0x36f06271f6a159b6626f55aa5b16cf3e74b4797789e8375911109a9ba9063154) | 20566325 |
| Allowlist `0x150C659d…` (test originator) | [`0x9cf53ccf…54feacdce355845`](https://scan.botchain.ai/tx/0x9cf53ccfa93ca3be11ad8d85b185940dd6539ea2fdff6831f54feacdce355845) | 20566327 |
| Mint mUSD → `0x68343Aa0…` | [`0xc16dd2d0…12cf40db1a8864db2`](https://scan.botchain.ai/tx/0xc16dd2d04e43400d42ab85019eb9b80d9a74110d740441712cf40db1a8864db2) | 20566330 |
| Mint mUSD → `0x150C659d…` | [`0xa0af3b73…841bb922b3a1d35ed1`](https://scan.botchain.ai/tx/0xa0af3b73a912eff851b8750a0918167a7d73a7e4444e44841bb922b3a1d35ed1) | 20566333 |
| Mint mUSD → `0x48D8f72D…` (investor) | [`0x65831e2f…b008405599b1796475`](https://scan.botchain.ai/tx/0x65831e2f52028a7eb484d4bb1dbf4d7ed48d08c0bcf618b008405599b1796475) | 20566335 |

17 transactions total, all confirmed in blocks 20566286–20566335.

### Test fixtures on mainnet

| Address | Role | Status |
|---|---|---|
| `0x68343Aa0598b7FCAA102769D172e59cdDfae10f2` | Deployer / `ARBITRATOR_1` | Allowlisted originator, minted 1,000,000 mUSD |
| `0x150C659d09fc5B9d53Cf2F18234517E8C702a614` | `ARBITRATOR_2` / test originator | Allowlisted originator, minted 1,000,000 mUSD |
| `0x48D8f72D20D8F9C0b7abab30AB846f4D82a08dc7` | `ARBITRATOR_3` / test investor | Minted 1,000,000 mUSD |

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
