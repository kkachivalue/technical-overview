# KkachiVal Technical Architecture

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

**Sample on-chain transactions** — live on the active contracts:

| Step | Transaction |
|---|---|
| Create market | [`0x9aa56655…b82427a2`](https://sepolia-explorer.giwa.io/tx/0x9aa56655812f08419545b542735adbf1d813528c84bea8fb43ec7d24b82427a2) (`createMarket`) |
| Open market | [`0xe7d190ed…5eb345a4`](https://sepolia-explorer.giwa.io/tx/0xe7d190edbf18bbd9e7b8f0eab138f8306b35c6c893541f5fd083a5a45eb345a4) (`openMarket`) |
| Settle trades | [`0xb7af3565…694dec20d`](https://sepolia-explorer.giwa.io/tx/0xb7af3565e17bbb2f112a0ba8275dee8b8eed2f53515864074c9c6bf694dec20d) (`settleComplementarySignedBatch`) |

**Testnet scope & production** — the beta runs operator-settled binary markets: off-chain
matching, on-chain settlement, a mock gUSDC collateral token, and finalize gated by an
admin-configurable delay currently set to zero (instant finalize). Lifecycle timing is
role-controlled (timestamps are on-chain metadata, not yet time-enforced), and the deployer
key still holds `DEFAULT_ADMIN`. Production wires canonical USDC, a multisig `DEFAULT_ADMIN`
with least-privilege roles, on-chain-enforced windows, and a non-zero finalization delay (§8).

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
| **GiwaMarketRegistry** | Market lifecycle (create → open → resolve → finalize); `finalizeMarket` gated by a global, admin-configurable **finalization delay** (`marketResolvedAt` + `challengePeriod`) — a time delay before finalize, not an on-chain dispute mechanism; `OPERATOR` / `RESOLVER` roles |
| **GiwaBinaryEngine** | CTF-style ERC-1155 outcome tokens; `split` / `merge` / `redeem`; per-market collateral backing ledger + mint-solvency guard; payout-numerator cap and a `finalized` gate on `redeem`; rejects zero-collateral fills; blocks mint/transfer into a resolved market; operator-triggered maintenance pause for incident response |
| **GiwaExchange** | EIP-712 signed orders, operator-settled — single (`settleSigned`, `settleComplementarySigned`, `settleComplementaryMergeSigned`) and batched (`settleSignedBatch`, `settleComplementarySignedBatch`) for settlement throughput under a rate-limited node; EIP-1271 smart-wallet signatures; per-market taker-fee cap with maker/taker discount |
| **GiwaUSDC** | Collateral token — a testnet mock stand-in, not a protocol contract; mainnet integrates canonical USDC |

- **Roles.** `DEFAULT_ADMIN` / `OPERATOR` (open, settle) / `RESOLVER` (report payouts) are
  distinct roles. At deploy the `RESOLVER` and `OPERATOR` roles are granted to dedicated keys
  and revoked from the deployer; settlement runs from a separate operator EOA. On the current
  testnet the deployer EOA still holds `DEFAULT_ADMIN` on all contracts and re-holds operator
  authority to run beta market operations, so role assignments are reversible by that single
  key. Production moves `DEFAULT_ADMIN` to a multisig and applies least-privilege separation (§8).
- **Collateral conservation.** Per-market `backing[conditionId]` with a mint-solvency guard;
  the split path credits the *actually received* collateral amount; CTF transfer semantics as a
  hard backstop. Fuzz-invariant tested (§6).

---

## 3. Services (Go)

Independently deployable tiers (each is its own container), so a tier can be scaled or moved
without touching the others.

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
- **EIP-712 + EIP-1271.** Orders are signed off-chain (gasless placement/cancel), domain-bound.
  Replay and overfill are bounded by cumulative fill accounting per order hash (not a single-use
  nonce); a trader can invalidate every order sharing a nonce via on-chain nonce cancellation.
  EIP-1271 lets smart wallets (e.g. Safe) trade under the same path via `STATICCALL` checks.

---

## 5. Security model

Threats and on-chain / protocol mitigations:

| Threat | Mitigation |
|---|---|
| Operator drains funds | Operator never custodies; funds live in the engine/CTF; `settleSigned` only moves value between the two signed parties; role-separated keys |
| Under-collateralized mint | Per-market backing ledger + mint-solvency guard; split path credits actually-received collateral; CTF transfer as a hard backstop; validated by a collateral-conservation invariant campaign (§6) |
| Double-spend in fill→settle window | Collateral/shares reserved at match (user-global, cross-market pending guards); released on settlement failure |
| Signature forgery / replay | EIP-712 domain-bound signing; cumulative fill accounting per order hash; on-chain nonce cancellation for bulk invalidation; EIP-1271 `STATICCALL` verification |
| Resolution manipulation | Separated `RESOLVER` role; `finalizeMarket` is gated by a global, admin-configurable finalization delay measured from `marketResolvedAt`. This is a time delay, **not** an on-chain dispute mechanism — there is no dispute/correction function, and the admin can change the global delay. On the testnet the delay is 0 (instant finalize); production sets it non-zero and moves admin to a multisig (§8) |
| Premature / mis-sized redeem | `redeem` is gated on a `finalized` flag (set only after the delay elapses); a payout-numerator cap bounds any single condition's payout |
| Zero-collateral / dust fills | Fills whose `price × quantity` rounds below 1 gUSDC unit are rejected on-chain, so a settle can't mint value against zero backing |
| Lifecycle timing | Trading/resolution timestamps are recorded on-chain as metadata; state transitions are role-controlled and not yet time-enforced on-chain. Production hardening enforces the windows on-chain (§8) |
| Doomed / griefing settlements | Pre-broadcast simulation (a reverting settle never mines); operator submit is idempotent + reconciled; transient RPC rate-limits (429) are absorbed with backoff-retry |
| Incident response | Operator-triggered maintenance pause on the engine halts mint/transfer paths without touching fund-recovery, for a controlled stop during an incident |

The three protocol contracts (Exchange, Registry, BinaryEngine) are source-verified on the
GIWA Sepolia explorer. Collateral on testnet is a mock gUSDC (open mint, gated to the testnet
chain id) — a stand-in, not a protocol contract, and intentionally left unverified because it
is replaced by canonical USDC on mainnet (§8).

---

## 6. Execution

- **Nonce safety** — per-EOA in-process submit gate + single-leader lock ⇒ sequential
  nonces from `PendingNonceAt`. Deployer and settler run on separate EOAs so market
  creation never contends with the settlement stream.
- **Settlement resilience** — each settle is simulated before broadcast (a doomed tx fails
  pre-broadcast instead of mining a reverted tx and burning gas); transient-nonce
  self-heal; gas preflight + treasury auto-refill.
- **Scaling** — the on-chain worker tier is a separate service from the API; the horizontal
  throughput path is market-sharded operator EOAs with independent nonce streams.
- **Observability** — every on-chain broadcast logs op + nonce + target; the worker leader
  logs its build identity on lock acquisition.
- **Testing** — 74 Foundry tests (unit, batch-settlement coverage, and 3 fuzz functions at
  10,000 runs each). Collateral conservation is a stateful invariant (`backing` sums,
  pool-covers-backing, supply-matches-backing) validated in a campaign of **10,000 runs ×
  500 call depth ≈ 5,000,000 contract calls per invariant**. A Go suite covers the matcher,
  settlement, and read model.
- **Reproducibility** — contracts compiled with `solc 0.8.35`, optimizer 200, EVM `osaka`,
  and deployed from a tagged source commit; constructor arguments are recorded in the deploy
  broadcast manifest. Full Foundry run logs, CI summary, and the deployment manifest are
  available to reviewers on request.

---

## 7. Why GIWA

- **Self-custody UX on a fast L2.** The whole point of the hybrid design — sign off-chain,
  settle on-chain — needs cheap, quick finality. GIWA's OP-Stack L2 gives low fees and fast
  confirmation, so a signed order settles without the trader ever leaving self-custody.
- **Canonical bridge for real collateral.** On mainnet, collateral is real USDC brought onto
  the L2 through GIWA's canonical bridge, replacing the testnet mock (§8).
- **Smart-wallet ready.** The Exchange already verifies EIP-1271 signatures, so GIWA-native
  smart wallets can trade on the same signed-order path as EOAs, no separate flow.
- **GIWA Wallet & faster confirmation (planned).** GIWA Wallet integration is
  integration-ready via the EIP-1271 path and planned to land once the wallet ships; we also
  plan to lean on GIWA's fast block confirmation (Flashblocks) to tighten the sign→settle loop,
  and a paymaster path (§8) to remove first-trade gas friction. These are planned, not yet live.
- **User flow.** discover → sign → settle → redeem → return: a trader finds a market, signs an
  order, sees it settle on-chain, redeems winnings after finalize, and returns to browse.

GIWA connect / wallet / Flashblocks docs: https://docs.giwa.io

---

## 8. What's next

- **Mainnet hardening** — external real USDC (not the mock gUSDC), `DEFAULT_ADMIN` moved to
  a multisig with least-privilege role separation, treasury fee recipient, a non-zero
  finalization delay, and on-chain-enforced trading/resolution windows. (Payout cap,
  `finalized`-gated redeem, zero-fill rejection and the engine maintenance pause are already
  in the contract build.)
- **Canonical USDC over GIWA's bridge** — collateral moves from the testnet mock to real
  USDC bridged onto the L2 through GIWA's canonical bridge.
- **On-chain dispute path** — the current finalization delay becomes a real challenge window:
  per-market immutable delay snapshotted at resolve time, plus a dispute / resolution-escalation
  function (today resolution rests on the `RESOLVER` role).
- **Gasless onboarding** — a paymaster / permit-relay path so a new user can approve and
  trade without first holding native gas.
- **Horizontal settlement** — market-sharded operator EOAs with independent nonce streams,
  so settlement throughput scales with load instead of serializing behind one signer.
- **Multi-outcome, neg-risk & combo bets** — categorical / bucketed markets, capital-efficient
  mutually-exclusive (neg-risk) settlement, and parlay-style combo positions that stake across
  several markets in one bet.

