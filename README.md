# Kkachival — Technical Architecture & Execution

Binary prediction market on **GIWA** (OP-Stack L2). **Hybrid order book**: matching is
off-chain (price-time priority); custody, collateral, and settlement are on-chain.
Positions are ERC-1155 outcome tokens collateralized in gUSDC. Live on GIWA Sepolia with
immutable, explorer-verified contracts.

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
| **GiwaMarketRegistry** | Market lifecycle (create → open → resolve → finalize), configurable challenge period, `OPERATOR`/`RESOLVER` roles |
| **GiwaBinaryEngine** | CTF-style ERC-1155 outcome tokens; `split`/`merge`/`redeem`; per-market collateral backing ledger + mint-solvency guard; blocks mint/transfer into a resolved market |
| **GiwaExchange** | EIP-712 signed orders, operator-settled (`settleSigned`, batch, complementary-merge); EIP-1271 smart-wallet signatures; per-market taker-fee cap |
| **GiwaUSDC** | Collateral token (testnet mock; mainnet wires external USDC) |

- `DEFAULT_ADMIN` / `OPERATOR` (settle, create) / `RESOLVER` (report payouts) are distinct
  roles; the deployer's operator/resolver roles are revoked at deploy time.
- On-chain collateral conservation: per-market `backing[conditionId]` with a mint-solvency
  guard; CTF transfer semantics as a hard backstop. Fuzz-invariant tested.

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
| Operator drains funds | Operator never custodies; funds live in the engine/CTF; `settleSigned` only moves value between the two signed parties; role-separated keys |
| Under-collateralized mint | Per-market backing ledger + mint-solvency guard; CTF transfer as a hard backstop; fuzz-invariant over millions of calls |
| Double-spend in fill→settle window | Collateral/shares reserved at match (user-global, cross-market pending guards); released on settlement failure |
| Signature forgery / replay | EIP-712 domain-bound signing, single-use order nonces, EIP-1271 `STATICCALL` verification |
| Resolution manipulation | Separated `RESOLVER` role + configurable challenge period before `finalize` |
| Doomed / griefing settlements | Pre-broadcast simulation (a reverting settle never mines); operator submit is idempotent + reconciled |

Contract source is verified on-chain. Testnet uses a mock gUSDC (open mint, gated to the
testnet chain id); mainnet wires an external real USDC.

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
- **Testing** — 71 Foundry tests (incl. collateral-conservation fuzz-invariants); Go suite
  over matcher, settlement, and read model.

---

## 7. Roadmap

- **Mainnet hardening gate** — external real USDC (not the mock), `DEFAULT_ADMIN` moved to
  a multisig, non-zero challenge period, treasury fee recipient, verified role revokes.
- **Horizontal settlement** — market-sharded multiple operator EOAs for parallel,
  independent nonce streams under load.
- **Multi-outcome & NegRisk** — categorical / bucketed markets and capital-efficient
  mutually-exclusive (neg-risk) settlement; taxonomy specified, deferred to a later epic.

---

## 8. Deployment (GIWA Sepolia)

`solc 0.8.35`, EVM `osaka`, optimizer 200; verified on `sepolia-explorer.giwa.io`.

| Contract | Address |
|---|---|
| GiwaExchange | `0x3baaa1fb331f1f6ea0520eb87ef54a6265acdb5b` |
| GiwaMarketRegistry | `0xd9e14c71cf7602b284747d172e3b3a3125b47172` |
| GiwaBinaryEngine | `0x47aa49969f44a6eeb352be7585e31cacf90a59e3` |
| gUSDC | `0x96aea16b9dadcce2052771375383c6a2cebad2f7` |
