# Kkachival — Technical Architecture & Execution

Binary prediction market on **GIWA** (OP-Stack L2). **Hybrid order book**: matching is
off-chain (price-time priority); custody, collateral, and settlement are on-chain.
Positions are ERC-1155 outcome tokens collateralized in gUSDC. Live on GIWA Sepolia with
immutable, explorer-verified protocol contracts.

---

## At a glance

**Live testnet beta** — https://beta.kkachival.com  ·  GIWA Sepolia (chain id 91342)  ·  explorer `sepolia-explorer.giwa.io`

**Verified contracts** — the three protocol contracts are source-verified on the GIWA Sepolia explorer (click to view source). These are the addresses the live beta runs on today.

| Contract | Address | Status |
|---|---|---|
| GiwaExchange | [`0x9f2a5c24e61e9be6243ec9951a78ea949ead77c8`](https://sepolia-explorer.giwa.io/address/0x9f2a5c24e61e9be6243ec9951a78ea949ead77c8) | ✅ verified |
| GiwaMarketRegistry | [`0xd9e14c71cf7602b284747d172e3b3a3125b47172`](https://sepolia-explorer.giwa.io/address/0xd9e14c71cf7602b284747d172e3b3a3125b47172) | ✅ verified |
| GiwaBinaryEngine | [`0x47aa49969f44a6eeb352be7585e31cacf90a59e3`](https://sepolia-explorer.giwa.io/address/0x47aa49969f44a6eeb352be7585e31cacf90a59e3) | ✅ verified |
| gUSDC (collateral) | [`0x96aea16b9dadcce2052771375383c6a2cebad2f7`](https://sepolia-explorer.giwa.io/address/0x96aea16b9dadcce2052771375383c6a2cebad2f7) | testnet mock — replaced by canonical USDC on mainnet |

**Sample on-chain transactions** — a market's full lifecycle, live on the active contracts:

| Step | Transaction |
|---|---|
| Create market | [`0x9aa56655…b82427a2`](https://sepolia-explorer.giwa.io/tx/0x9aa56655812f08419545b542735adbf1d813528c84bea8fb43ec7d24b82427a2) (`createMarket`) |
| Open market | [`0xe7d190ed…5eb345a4`](https://sepolia-explorer.giwa.io/tx/0xe7d190edbf18bbd9e7b8f0eab138f8306b35c6c (`openMarket`) |
| Settle trades | [`0xb7af3565…694dec20d`](https://sepolia-explorer.giwa.io/tx/0xb7af3565e17bbb2f112a0ba8275dee8b8eed2f53515864074c9c6bf694dec20d) (`settleComplementarySignedBatch`) |

**Testnet scope & production** — the beta runs operator-settled binary markets: off-chain
matching, on-chain settlement, **instant finalize (challenge period = 0)**, and a mock
gUSDC collateral token. The dispute/challenge window is implemented in the contracts but
set to zero on the testnet; production wires canonical USDC, a multisig `DEFAULT_ADMIN`, a
non-zero challenge period, and a treasury fee recipient (§7).

---

## 1. Architecture

```
       EIP-712 signed order                         operator settles on-chain
 ┌────────┐  ──────────────►  ┌────────┐  match   ┌──────────┐  settleSigned  ┌───────────────┐
 │ Client │                   │  API   │ ───────► │ Matcher  │ ─────────────► │  Contracts    │
 │ (web)  │ ◄──────────────   │ HTTP+WS│  fills   │ off-chain│                │ Exchange /    │
 └────────┘   WS push         └────────┘  (Redis) │  book    │                │ Registry /    │
      ▲                            ▲              └──────────┘                │ Engine / USDC │
      │                            │ read model                              └───────┬───────┘
      │                       ┌────────┐        on-chain events        ┌──────────┐  │ events
      └───────────────────────│  API   │ ◄───────────────────────────  │ Indexer  │ ◄┘
                              read model└────────┘   Postgres           └──────────┘
```

**Trade lifecycle**

1. Client signs an EIP-712 limit order (EOA or EIP-1271 smart wallet).
2. Matcher matches it off-chain by price-time priority, emits a fill.
3. Settlement worker submits `settleSigned` on-chain from an operator EOA.
4. Engine moves collateral and mints/transfers ERC-1155 outcome tokens.
5. Indexer applies the on-chain event to the Postgres read model.
6. API pushes the update to the client over WebSocket.

Collateral is reserved at order/match time (pending-settlement guards, user-global and
cross-market) and released on settlement failure — a trader cannot over-commit in the
fill→settle window.

---

## 2. Contracts (immutable, versioned redeploy — no proxy)

| Contract | Responsibility |
|---|---|
| **GiwaMarketRegistry** | Market lifecycle (create → open → resolve → finalize); `finalizeCondition` gated by a configurable per-market challenge period (`marketResolvedAt` + `challengePeriod`); `OPERATOR`/`RESOLVER` roles |
| **GiwaBinaryEngine** | CTF-style ERC-1155 outcome tokens; `split`/`merge`/`redeem`; per-market collateral backing ld; payout-numerator cap and a `finalized` gate on `redeem`; rejects zero-collateral fills; blocks mint/transfer into aresolved market; operator-triggered maintenance pause for incident response |
| **GiwaExchange** | EIP-712 signed orders, operator-settled — single (`settleSigned`, `settleComplementarySigned`, `gned`) and batched (`settleSignedBatch`, `settleComplementarySignedBatch`) for settlement throughput under a rate-limitednode; EIP-1271 smart-wallet signatures; per-market taker-fee cap with maker/taker discount |
| **GiwaUSDC** | Collateral token — a testnet mock stand-in, not a protocol contract; mainnet integrates canonical US

- `DEFAULT_ADMIN` / `OPERATOR` (settle, create) / `RESOLVER` (report payouts) are distinct
  roles; the deployer's operator/resolver roles are revoked at deploy time.
- On-chain collateral conservation: per-market `backing[conditionId]` with a mint-solvency
  guard; CTF transfer semantics as a hard backstop. Fuzz-invariant tested (§6).

---

## 3. Services (Go)

Independently deployable tiers; a load spike in one cannot take down another.

- **`api`** — HTTP + WebSocket, stateless request/serve tier.
- **`matcher`** — off-chain order-book engine; price-time matching; publishes fills.
- **`worker`** — background tier, single active instance via a Postgres advisory lock:
  settlement, market deployment, on-chain resolution/finalize, reconciliation, snapshots.
- **`indexer`** — streams on-chain events into the read model (positions, balances).

Processes are bridged by Redis streams (order results, balance/position updates,
settlement-failure notices), so the serve tier and the submit tier scale independently.

---

## 4. Design rationale & tradeoffs

- **Hybrid order book.** A fully on-chain CLOB is gas-prohibitive per order and too slow
  for quote/cancel churn; fully off-chain custody re-introduces counterparty trust. We
  match off-chain (fast, free placement/cancel) and settle on-chain (self-custody,
  verifiable). Users sign orders and never surrender keys; the operator can only settle
  fills that both parties signed, and never custodies funds. Cost: the operator can delay
  or censor a fill — bounded because signed orders can't be forged or altered, funds live
  in the engine (not the operator), and a failed settle releases the trader's reservation.
- **Immutable, no proxy.** An upgradeable proxy under a single operator is an upgrade-key
  rug vector; immutability is the neutrality guarantee a betting venue needs. Cost: patches
  require redeploy + state migration (pause → redeploy → migrate → cutover runbook) instead
  of a hot upgrade — accepted in exchange for credible neutrality.
- **CTF-style collateral.** 1 gUSDC `split`s into 1 YES + 1 NO ERC-1155; `merge` reverses;
  `redeem` pays out after resolution. A per-market `backing[conditionId]` cap means the
  engine can never mint more outcome value than the collateral it holds — solvency is an
  on-chain invariant, not an operator promise. Complementary SELL+SELL orders settle via a
  merge-cross path so both sides net out through the engine.
- **EIP-712 + EIP-1271.** Orders are signed off-chain (gasless placement/cancel) and
  domain-bound with single-use nonces (replay-safe). EIP-1271 lets smart wallets (e.g.
  Safe) trade under the same path via `STATICCALL` signature checks.

---

## 5. Security model

Threats and on-chain / protocol mitigations:

| Threat | Mitigation |
|---|---|
| Operator drains funds | Operator never custodies; funds live in the engine/CTF; `settleSigned` only moves value bets; role-separated keys |
| Under-collateralized mint | Per-market backing ledger + mint-solvency guard; CTF transfer as a hard backstop; validated by a collateral-conservation invariant campaign (§6) |
| Double-spend in fill→settle window | Collateral/shares reserved at match (user-global, cross-market pending guards)ailure |
| Signature forgery / replay | EIP-712 domain-bound signing, single-use order nonces, EIP-1271 `STATICCALL` verification |
| Resolution manipulation | Separated `RESOLVER` role; `finalizeCondition` is gated on-chain by a per-market challengt` + `challengePeriod`) so a reported payout cannot be redeemed until the window elapses. No on-chain dispute oracle(resolution is operator/`RESOLVER`-driven). **On the testnet beta the challenge period is set to 0 (instant finalize); production sets it non-zero (§7).** |
| Premature / mis-sized redeem | `redeem` is gated on a `finalized` flag (set only after the challenge window); a payny single condition's payout |
| Zero-collateral / dust fills | Fills whose `price × quantity` rounds below 1 gUSDC unit are rejected on-chain, so a settle can't mint value against zero backing |
| Doomed / griefing settlements | Pre-broadcast simulation (a reverting settle never mines); operator submit is idempent RPC rate-limits (429) are absorbed with backoff-retry so they never surface as a settle failure |
| Incident response | Operator-triggered maintenance pause on the engine halts mint/transfer paths without touching fund-recovery, for a controlled stop during an incident |

The three protocol contracts (Exchange, Registry, BinaryEngine) are source-verified on the
GIWA Sepolia explorer. Collateral on testnet is a mock gUSDC (open mint, gated to the
testnet chain id) — a stand-in, not a protocol contract, and intentionally not verified as
it is replaced by canonical USDC on mainnet.

---

## 6. Execution

- **Nonce safety** — per-EOA in-process submit gate + single-leader lock ⇒ sequential
  nonces from `PendingNonceAt`. Deployer and settler run on separate EOAs so market
  creation never contends with the settlement stream.
- **Settlement resilience** — each settle is simulated before broadcast (a doomed tx fails
  pre-broadcast instead of mining a reverted tx and burning gas); transient-nonce
  self-heal; gas preflight + treasury auto-refill.
- **Scaling** — the on-chain worker tier is a separate service from the API; horizontal
  throughput path is market-sharded multiple settler EOAs (independent nonce streams).
- **Observability** — every on-chain broadcast logs op + nonce + target; the worker leader
  logs its build identity on lock acquisition.
- **Testing** — 74 Foundry tests (unit, batch-settlement coverage, and 3 fuzz functions at
  10,000 runs each). Collateral conservation is a stateful invariant (`backing` sums,
  pool-covers-backing, supply-matches-backing) validated in a campaign of **10,000 runs ×
  500 call depth ≈ 5,000,000 contract calls per invariant** — the "millions of calls" that
  back the solvency guarantee. A Go suite covers the matcher, settlement, and read model.

---

## 7. Roadmap

- **Mainnet hardening gate** — external real USDC (not the mock), `DEFAULT_ADMIN` moved to
  a multisig, treasury fee recipient, a non-zero challenge period, and verified role
  revokes. (On-chain challenge-period finalize, payout-cap / `finalized`-gated redeem,
  zero-fill rejection and the engine maintenance pause are already in the contract build.)
- **Horizontal settlement** — market-sharded multiple operator EOAs for parallel,
  independent nonce streams under load.
- **Multi-outcome & NegRisk** — categorical / bucketed markets and capital-efficient
  mutually-exclusive (neg-risk) settlement; taxonomy specified, deferred to a later epic.

---

## 8. Deployment (GIWA Sepolia, chain id 91342)

`solc 0.8.35`, EVM `osaka`, optimizer 200. The protocol suite is source-verified on
`sepolia-explorer.giwa.io` — click an address for the source. These are the addresses the
live beta runs on today.

| Protocol contract | Address | Source |
|---|---|---|
| GiwaExchange | [`0x9f2a5c24…ad77c8`](https://sepolia-explorer.giwa.io/address/0x9f2a5c24e61e9be6243ec9951a78ea949ead77c8) | ✅ verified |
| GiwaMarketRegistry | [`0xd9e14c71…b47172`](https://sepolia-explorer.giwa.io/address/0xd9e14c71cf7602b284747d172e3b3a3125b47172) | ✅ verified |
| GiwaBinaryEngine | [`0x47aa4996…0a59e3`](https://sepolia-explorer.giwa.io/address/0x47aa49969f44a6eeb352be7585e31cacf90a59e3) | ✅ verified |

Collateral (testnet only): gUSDC [`0x96aea16b…bad2f7`](https://sepolia-explorer.giwa.io/address/0x96aea16b9dadcce2052771375383c6a2cebad2f7) — a mock stand-in, not a protocol contract; mainnet integrates canonical USDC.
