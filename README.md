# 0Swap: AI-Powered DeFi Swapper on Base / Dreamspace

**0Swap** is a decentralized finance (DeFi) application on the Base chain, enabling seamless token swaps via Uniswap V3. It features a React-based swap interface for manual trades and an AI-powered chat assistant (leveraging Grok from xAI) for natural language trading. Swaps support ETH and select ERC-20 tokens, with built-in fee handling and robust security.

---

## 🚀 Features

- **Decentralized Swaps:** Trustless, non-custodial token swaps on Base.
- **AI Chat Trading:** Use natural language to trade via Grok (xAI).
- **Manual UI:** React-based interface for direct, slippage-protected swaps.
- **Fee Management:** Configurable protocol fee; supports deduction from input or output.
- **Security:** Ownable admin, reentrancy guards, safe transfers, robust validation.

---

## 📜 Contract Addresses

| Contract                  | Address                                                                                 | Notes                               |
|---------------------------|-----------------------------------------------------------------------------------------|-------------------------------------|
| Main (EmptyContract_v1)   | `0xf67002e4e16DFCeDDf5d149A97dd7A3C94986178`                                            | Base Mainnet, verified on Basescan  |
| Uniswap V3 SwapRouter02   | `0x2626664c2603336E57B271c5C0b26F421741e481`                                            | All routing                         |
| WETH (Wrapped ETH)        | `0x4200000000000000000000000000000000000006`                                            | Base canonical                      |
| Quoter V2                 | `0x3d4e44Eb1374240CE5F1B871ab261CD16335B76a`                                            | For real-time pricing               |

---

## 🪙 Supported Tokens

0Swap currently supports the following Base Mainnet tokens (selected for simplicity and liquidity):

| Symbol | Name      | Address                                         | Decimals | Notes            |
|--------|-----------|-------------------------------------------------|----------|------------------|
| ETH    | ETH       | `0x0000000000000000000000000000000000000000`    | 18       | Native           |
| USDC   | USDC      | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA029136`   | 6        |                  |
| USDT   | USDT      | `0xfde4C96c8593536E31F229EA8f37b2ADa2699bb2`    | 6        |                  |
| cbBTC  | cbBTC     | `0xcbB7C0000aB88B473b1f5aFd9ef808440eed33Bf8`   | 8        | Bridged BTC      |

- **Pairs:** All token combinations supported (e.g., ETH ↔ USDC, USDC ↔ USDT).
- **Pool Fees:** Auto-selects optimal Uniswap V3 fee tier (`0.01%`, `0.05%`, `0.3%`, `1%`) based on liquidity. Manual override available.

---

## 💸 Protocol Fee

- **Default Fee:** 0.1% (10 BPS) — configurable (0–100%) by admin.
- **Collection:** Sent to `feeCollector` (default: `0xB9DCA4c38547Aea56C1036aa421b4DA083e62bcc`).
- **Modes:**
  - **From Input (default):** Deducted before swap. (e.g., input 100 USDC → 99.9 swapped, 0.1 to collector)
  - **From Output:** Deducted after swap (exact-output mode required). (e.g., output 100 USDC → 100.1 gross, 0.1 to collector, 100 to user)

- **Admin Functions:** `setFeeBPS`, `setFeeCollector`, `setFeeFromOutput`
- **Events:** `FeeCollected` logs each protocol deduction.

---

## 🔄 How Transactions Are Routed

All swaps are routed through Uniswap V3's `SwapRouter02` for optimal pricing/liquidity. The contract acts as a wrapper:

### Input Handling

- **ERC-20:** Uses `safeTransferFrom` from user to contract, then approves router.
- **ETH:** Native ETH is deposited as WETH, then approved.

### Routing

- **Exact Input:** Calls `exactInputSingle` (tokenIn, tokenOut, pool fee, amountIn, minOut, recipient=contract)
- **Exact Output:** Calls `exactOutputSingle` (amountOut, maxIn, refund excess)
- **ETH Output:** Unwraps WETH to ETH with `_handleETHTransfer` (pull-payment for failed sends)
- **Fee Deduction:** Pre-swap or post-swap, per mode. (Exact-output swaps require `feeFromOutput=true`)

### Output Handling

- **Transfer:** Sends output tokens/ETH to user (post-fee if from output mode).
- **Slippage:** `amountOutMinimum` (default 1% buffer via UI)
- **Deadline:** 20 minutes by default, up to 30 min max
- **Gas:** Estimated with 20% safety buffer

### Quoting

- Uses Quoter V2 for `quoteExactInputSingle` / `quoteExactOutputSingle` (view-only, no execution)

#### Example Flow: *Exact Input USDC → ETH*

1. User approves & transfers 100 USDC.
2. Fee (0.1%) deducted: 0.1 USDC to collector, 99.9 USDC to router.
3. Router swaps via 0.3% pool → ~0.025 ETH (unwrapped if needed).
4. 0.025 ETH sent to user.

---

## 🛡️ Security

- **Access Control:** `Ownable` for admin functions (fee updates, quoter address)
- **Reentrancy:** `ReentrancyGuard` on all swap/withdraw functions
- **Transfers:** `SafeERC20` for ERC-20 (prevents silent failures)
- **Input Validation:**
  - Non-zero amounts/addresses
  - Only valid Uniswap fee tiers
  - Disallows same-token swaps
  - Deadline within 1–30 minutes
  - Fee capped at ≤100% BPS
  - Reverts on insufficient output

- **ETH Safety:** Failed sends queued in `pendingETHWithdrawals`; can withdraw via `withdrawPendingETH`
- **Dependencies:** Audited OpenZeppelin contracts; only external call is to Uniswap
- **Recommendations:** Monitor for upgrades, use hardware wallets, set slippage <5%
- **Vulnerability Status:** No known vulnerabilities as of September 20, 2025; contract verified on Basescan.

---

## 💻 Example Usage

### Manual Swap (Web UI)

1. Connect MetaMask (UI will prompt for Base network if needed)
2. Select "From: USDC", enter amount (e.g., 100)
3. Select "To: ETH", view live quote (e.g., ~0.025 ETH)
4. Adjust slippage (default 1%) and deadline (default 20 min)
5. Click "Swap" (UI will prompt for approval if needed, then execute)
6. View transaction on Basescan

### AI Chat Commands

- **Buy:**  
  `Buy 20 USDT`  
  → Select input (e.g., ETH) → View quote (~0.008 ETH) → Confirm → Swap
- **Sell:**  
  `Sell 10 USDC`  
  → Auto-route to ETH → View quote (~0.0025 ETH) → Confirm → Swap
- **Swap:**  
  `Swap 1 ETH to cbBTC`  
  → View quote → Confirm → Execute
- **Other:**  
  `What's the best strategy for stablecoin yields?`  
  → AI provides a natural language response

- **Transaction Example:**  
  Search [0xf67002e4e16DFCeDDf5d149A97dd7A3C94986178 on Basescan](https://basescan.org/address/0xf67002e4e16DFCeDDf5d149A97dd7A3C94986178) for live swap events (`SwapExecuted` with full details).

---

## 📝 License & Credits

- Core contracts: [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts)
- Routing: [Uniswap V3](https://uniswap.org/)
- Chat AI: [Grok by xAI](https://x.ai/)
- UI: Custom React (see `/ui`)
- Base chain: [base.org](https://base.org/)

---

## Acknowledgements

- Dreamspace and space and time db for the infrastracture.
- Uniswap, Base, OpenZeppelin, and xAI for foundational tooling and protocols.
- Community feedback and users — keep swapping and keep building!

---

## 📫 Questions & Support

- Open an [issue](https://github.com/0xfdbu/0swap/issues) or [pull request](https://github.com/0xfdbu/0swap/pulls)
