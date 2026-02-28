# ⚓ Anchor AMM — Solana Automated Market Maker

A decentralized **Automated Market Maker (AMM)** built on Solana using the [Anchor framework](https://www.anchor-lang.com/). Implements a constant-product curve (`x * y = k`) for trustless token swaps, liquidity provisioning, and LP token management — fully on-chain.

---

## 📐 Architecture Overview

```mermaid
graph TD
    Client["🖥️ Client / dApp<br/>TypeScript + Anchor SDK"]
    Solana["⚡ Solana Localnet<br/>Program ID: Abckz5L...rLWN"]
    Program["📦 AMM Program<br/>lib.rs"]

    Initialize["🏗️ initialize<br/>seed · fee · authority"]
    Deposit["💰 deposit<br/>add liquidity · mint LP"]
    Withdraw["🏧 withdraw<br/>remove liquidity · burn LP"]
    Swap["🔄 swap<br/>is_x · amount_in · min_out"]

    State["🗃️ Config PDA<br/>seed · fee · locked<br/>mint_x · mint_y<br/>config_bump · lp_bump"]

    Client -->|RPC calls| Solana
    Solana --> Program
    Program --> Initialize
    Program --> Deposit
    Program --> Withdraw
    Program --> Swap
    Initialize -->|creates| State
    Deposit -->|reads| State
    Withdraw -->|reads| State
    Swap -->|reads| State
```

---

## 🏦 On-Chain Account Structure

```mermaid
graph TD
    Wallet["👤 Initializer Wallet<br/>Payer + Signer"]

    Config["🔑 Config PDA<br/>seeds: b'config' + seed<br/>─────────────────────<br/>seed · fee · locked<br/>mint_x · mint_y<br/>config_bump · lp_bump"]

    VaultX["🏦 Vault X ATA<br/>holds Token X<br/>authority = Config"]
    VaultY["🏦 Vault Y ATA<br/>holds Token Y<br/>authority = Config"]
    MintLP["🪙 LP Mint PDA<br/>seeds: b'lp' + config.key<br/>authority = Config<br/>decimals = 6"]

    Wallet -->|initialize| Config
    Config -->|owns| VaultX
    Config -->|owns| VaultY
    Config -->|mint authority| MintLP
```

---

## 💰 Deposit Flow

```mermaid
flowchart TD
    A(["👤 deposit<br/>amount_lp · max_x · max_y"])
    B{Pool locked?}
    C["❌ PoolLocked"]
    D{Pool empty?<br/>supply = 0<br/>vault_x = 0<br/>vault_y = 0}
    E["x = max_x<br/>y = max_y"]
    F["ConstantProduct::<br/>xy_deposit_amounts_from_l()<br/>x·y proportional to k"]
    G{x ≤ max_x<br/>AND y ≤ max_y?}
    H["❌ SlippageExceeded"]
    I["CPI: transfer<br/>user_x ──► vault_x"]
    J["CPI: transfer<br/>user_y ──► vault_y"]
    K["CPI: mint_to<br/>LP tokens ──► user_lp"]
    L(["✅ Done"])

    A --> B
    B -->|YES| C
    B -->|NO| D
    D -->|YES| E
    D -->|NO| F
    E --> G
    F --> G
    G -->|NO| H
    G -->|YES| I
    I --> J
    J --> K
    K --> L
```

---

## 🔄 Swap Flow

```mermaid
flowchart TD
    A(["👤 swap<br/>is_x · amount_in · min_out"])
    B{Pool locked?}
    C["❌ PoolLocked"]
    D{LP supply = 0?}
    E["❌ NoLiquidityInPool"]
    F["ConstantProduct::init<br/>vault_x · vault_y · supply · fee"]
    G["c.swap<br/>LiquidityPair::X or Y<br/>calculates res.deposit<br/>and res.withdraw"]
    H["CPI: transfer<br/>user_in ──► vault_in<br/>amount = res.deposit"]
    I["CPI: transfer w/ signer<br/>vault_out ──► user_out<br/>amount = res.withdraw"]
    J(["✅ Done"])

    A --> B
    B -->|YES| C
    B -->|NO| D
    D -->|YES| E
    D -->|NO| F
    F --> G
    G --> H
    H --> I
    I --> J
```

---

## 🏧 Withdraw Flow

```mermaid
flowchart TD
    A(["👤 withdraw<br/>amount_lp · min_x · min_y"])
    B{Pool locked?}
    C["❌ PoolLocked"]
    D["ConstantProduct::<br/>xy_withdraw_amounts_from_l()<br/>calculate proportional x, y"]
    E{x ≥ min_x<br/>AND y ≥ min_y?}
    F["❌ SlippageExceeded"]
    G["CPI: burn<br/>LP tokens from user_lp"]
    H["CPI: transfer w/ signer<br/>vault_x ──► user_x"]
    I["CPI: transfer w/ signer<br/>vault_y ──► user_y"]
    J(["✅ Done"])

    A --> B
    B -->|YES| C
    B -->|NO| D
    D --> E
    E -->|NO| F
    E -->|YES| G
    G --> H
    H --> I
    I --> J
```

---

## 🔁 CPI & PDA Signer Pattern

```mermaid
sequenceDiagram
    participant User
    participant AMM as AMM Program
    participant Config as Config PDA
    participant SPL as SPL Token Program

    Note over User,SPL: Deposit — user signs directly
    User->>AMM: deposit(amount, max_x, max_y)
    AMM->>SPL: transfer(user_x → vault_x, authority=user)
    AMM->>SPL: transfer(user_y → vault_y, authority=user)
    AMM->>SPL: mint_to(user_lp, authority=Config PDA)
    Config-->>SPL: signs via signer_seeds ["config", seed, bump]
    SPL-->>User: LP tokens received ✅

    Note over User,SPL: Swap — vault out requires PDA signer
    User->>AMM: swap(is_x, amount_in, min_out)
    AMM->>SPL: transfer(user_in → vault_in, authority=user)
    AMM->>SPL: transfer(vault_out → user_out, authority=Config PDA)
    Config-->>SPL: signs via signer_seeds ["config", seed, bump]
    SPL-->>User: output tokens received ✅
```

---

## 📁 Project Structure

```
anchor-amm/
├── Anchor.toml                  # Workspace config, cluster, wallet
├── Cargo.toml                   # Workspace root
├── package.json                 # Anchor SDK, Mocha, Chai
├── tsconfig.json                # TypeScript config for tests
├── rust-toolchain.toml          # Pinned Rust 1.89.0
│
├── migrations/
│   └── deploy.ts                # Anchor deploy script
│
├── programs/amm/src/
│   ├── lib.rs                   # Entrypoint — routes all 4 instructions
│   ├── constants.rs             # Program-wide constants
│   ├── error.rs                 # Custom AmmError codes
│   │
│   ├── instructions/
│   │   ├── initialize.rs        # Pool init — Config PDA + vaults + LP mint
│   │   ├── deposit.rs           # Add liquidity, mint LP tokens
│   │   ├── withdraw.rs          # Remove liquidity, burn LP tokens
│   │   └── swap.rs              # Token swap via constant product curve
│   │
│   └── state/
│       └── config.rs            # Config account definition
│
└── tests/
    └── amm.ts                   # Integration tests (Mocha + Anchor)
```

---

## 🛠️ Instructions

### `initialize(seed, fee, authority)`
| Account | Role |
|---|---|
| `initializer` | Payer + signer |
| `mint_x / mint_y` | The two tokens forming the pair |
| `config` | PDA — central pool state |
| `mint_lp` | PDA — LP token mint (authority = config) |
| `vault_x / vault_y` | ATAs holding pool reserves |

### `deposit(amount, max_x, max_y)`
| Param | Description |
|---|---|
| `amount` | LP tokens to mint |
| `max_x` | Max Token X to deposit (slippage guard) |
| `max_y` | Max Token Y to deposit (slippage guard) |

### `withdraw(amount, min_x, min_y)`
| Param | Description |
|---|---|
| `amount` | LP tokens to burn |
| `min_x` | Min Token X expected back (slippage guard) |
| `min_y` | Min Token Y expected back (slippage guard) |

### `swap(is_x, amount_in, min_amount_out)`
| Param | Description |
|---|---|
| `is_x` | `true` = sell Token X, `false` = sell Token Y |
| `amount_in` | Amount of input token to sell |
| `min_amount_out` | Minimum output expected (slippage guard) |

---

## 🔐 Config Account

```rust
pub struct Config {
    pub seed: u64,                 // Unique pool identifier
    pub authority: Option<Pubkey>, // Optional admin (can lock pool)
    pub mint_x: Pubkey,            // Token X mint address
    pub mint_y: Pubkey,            // Token Y mint address
    pub fee: u16,                  // Swap fee in basis points (30 = 0.3%)
    pub locked: bool,              // Emergency lock flag
    pub config_bump: u8,           // PDA bump for config
    pub lp_bump: u8,               // PDA bump for LP mint
}
```

---

## 🧩 Core Concepts

**Constant Product Curve (`x * y = k`)** — The invariant governing all swaps and liquidity ops. The `constant-product-curve` crate handles all math — deposit ratios, swap output amounts, and fees.

**Config PDA** — Central pool account derived from a `seed`. Multiple independent pools can coexist by using different seeds.

**LP Tokens** — Minted to liquidity providers proportional to their pool share. Burning LP tokens returns the underlying assets.

**Slippage Protection** — Every instruction takes a min/max bound. If the pool ratio moves beyond tolerance, the transaction reverts.

**PDA Vault Authority** — Both vaults are ATAs owned by Config PDA. The program signs outgoing transfers using `CpiContext::new_with_signer` with seeds `["config", seed, bump]` — no private key needed.

---

## 🚀 Getting Started

### Prerequisites
- [Rust](https://www.rust-lang.org/tools/install) — pinned to `1.89.0`
- [Solana CLI](https://docs.solana.com/cli/install-solana-cli-tools)
- [Anchor CLI](https://www.anchor-lang.com/docs/installation) `v0.32.1`
- [Yarn](https://yarnpkg.com/)

```bash
git clone https://github.com/Subodhkd001/anchor-amm.git
cd anchor-amm
yarn install
anchor build
anchor test
```

---

## 🧪 Tech Stack

| Layer | Technology |
|---|---|
| Smart Contract | Rust + Anchor `0.32.1` |
| Curve Math | `constant-product-curve` |
| Token Standard | SPL Token (`anchor-spl`) |
| Tests | TypeScript + Mocha + Chai |
| Network | Solana Localnet |

---

## 📝 License

ISC
