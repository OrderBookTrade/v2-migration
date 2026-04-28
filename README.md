# Polymarket CLOB V2 Migration Guide



If your bot manages real capital, the Polymarket CLOB v2 migration is not a documentation task.

***It is a risk-control task.*** 

Polymarket CLOB v2 changes more than the client package. 

[The official migration guide](https://docs.polymarket.com/v2-migration) describes this as a coordinated upgrade across **new Exchange contracts**, a **rewritten CLOB backend**, and a **new collateral token, pUSD**. 


The cutover is scheduled for April 28, 2026 around (11:00 UTC), with roughly one hour of downtime, and all open orders are expected to be wiped during the cutover.  



**That means**

**builders**, **makers**, and **API traders** need to re-check the full trading path:

```
wallet -> signature -> collateral -> allowance -> order -> match -> settlement -> position
```



These are the failure modes that cause makers to lose money during exchange migrations:

* Orders may submit, but never become truly live.

* WebSocket fills may appear before settlement is durable.

* Collateral balance may be checked on the wrong wallet.

* PUSD allowance may be granted to the wrong contract.

  

So  this guide focuses on the parts that actually break in production 

> * **Proxy wallets (EIP1271 Support)** 
> * **Signature types**
> * **PUSD migration**
> * **Order lifecycle**
> * **WebSocket ordering**
> * **Ghost fills** 
> * **Reconciliation**



This is a comprehensive guide for migrating from Polymarket CLOB v1 to Polymarket CLOB v2.



## TL;DR

Do not treat CLOB v2 migration as *update the SDK and restart the bot.*



At minimum, verify below setups

> - v2 client package and host configuration
> - wallet type and `signatureType`
> - pUSD collateral path
> - USDC.e → pUSD wrapping
> - balance and allowance from the actual trading address
> - builder code configuration
> - order submission and rejection behavior
> - WebSocket order updates
> - REST open orders and fills
> - settlement status
> - position delta
> - cancel behavior
> - kill switch behavior



**And a safe migration path is:**

> 1. isolate and cancel v1 orders
> 2. install and pin the v2 SDK( [Typescript](https://github.com/Polymarket/clob-client-v2) [Python](https://github.com/Polymarket/py-clob-client-v2) [Rust](https://github.com/Polymarket/rs-clob-client-v2))
> 3. configure the v2 client
> 4. verify collateral, balance, and allowance
> 5. place a tiny canary order
> 6. track `submit -> accepted -> matched -> mined -> confirmed`
> 7. reconcile WebSocket, REST, and on-chain state
> 8. keep size small during the launch window
> 9. increase size only after repeated successful settlement



## **1. What CLOB V2 Actually Changes**



V2  change the [`on-chain exchange contracts`]( https://github.com/Polymarket/ctf-exchange-v2), the `EIP-712 signing domain`, the `on-chain order struct`, the `collateral token`, the `builder attribution mechanism` and the `SDK package` and `constructor shape`, all in a single coordinated cutover. 



**Most V1 clients break because every signed order they produce will fail validation against the new exchange contract** 



For a trading bot, these changes touch the entire execution pipeline.

| Component                           | V1                                                           | V2                                                           |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **CTF Exchange**                    | 0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E                   | 0xE111180000d2663C0091e4f400237545B87B996B                   |
| **Neg Risk CTFExchange**            | 0xC5d563A36AE78145C45a50134d48A1215220f80a                   | 0xe2222d279d744050d28e00520010520000310F59                   |
| **EIP-712 Exchange domain version** | "1"                                                          | "2"                                                          |
| **SDK Package**                     | @polymarket/clob-client<br />py-clob-client<br />rs-clob-client | @polymarket/clob-client-v2<br />py-clob-client-v2<br />rs-clob-client-v2 |
| **Host**                            | https://clob.polymarket.com                                  | https://clob-v2.polymarket.com<br /><br />updated to https://clob.polymarket.com after cutover |
| **Order struct**                    | nonce<br/>feeRateBps<br/>taker                               | timestamp<br/>metadata<br/>builder                           |



**Order struct change (signed payload, per migration docs):**

 V1's twelve-field struct (`salt, maker, signer, taker, tokenId, makerAmount, takerAmount, expiration, nonce, feeRateBps, side, signatureType`) becomes a ten-field V2 struct 

```solidity
//keccak256(
//     "Order(uint256 salt,address maker,address signer,uint256 tokenId,uint256 makerAmount,uint256 takerAmount,uint8
// side,uint8 signatureType,uint256 timestamp,bytes32 metadata,bytes32 builder)" );

struct Order {
    /// @notice Unique salt to ensure entropy
    uint256 salt;
    /// @notice Maker of the order, i.e the source of funds for the order
    address maker;
    /// @notice Signer of the order
    address signer;
    /// @notice Token Id of the CTF ERC1155 asset to be bought or sold
    /// If BUY, this is the tokenId of the asset to be bought, i.e the takerAssetId
    /// If SELL, this is the tokenId of the asset to be sold, i.e the makerAssetId
    uint256 tokenId;
    /// @notice Maker amount, i.e the maximum amount of tokens to be sold
    uint256 makerAmount;
    /// @notice Taker amount, i.e the minimum amount of tokens to be received
    uint256 takerAmount;
    /// @notice The side of the order: BUY or SELL
    Side side;
    /// @notice Signature type used by the Order: EOA, POLY_PROXY, POLY_GNOSIS_SAFE or POLY_1271
    SignatureType signatureType;
    /// @notice Unix timestamp in milliseconds at which the order was created
    uint256 timestamp;
    /// @notice The metadata associated with the order, hashed
    bytes32 metadata;
    /// @notice The builder code associated with the order, indicating its origin
    bytes32 builder;
    /// @notice The order signature
    bytes signature;
}
```



**Below is an example to build V2 Clob Client**



```typescript
// npm install @polymarket/clob-client-v2 viem
import { ClobClient, Chain, OrderType, Side } from "@polymarket/clob-client-v2";
import { createWalletClient, http } from "viem";
import { privateKeyToAccount } from "viem/accounts";

const POLYMARKET_CLOB_CLIENT = "<polymarket-clob-host>"
const account = privateKeyToAccount("0x..."); // pk from dotenv
const walletClient = createWalletClient({ account, transport: http() });

// L1 phase: derive API creds
const bootstrap = new ClobClient({
  host: POLYMARKET_CLOB_CLIENT,
  chain: Chain.POLYGON,
  signer: walletClient,
});
const creds = await bootstrap.createOrDeriveApiKey();

// L2 phase: full-auth client for trading
const client = new ClobClient({
  host: POLYMARKET_CLOB_CLIENT,
  chain: Chain.POLYGON,
  signer: walletClient,
  creds,
```

* The constructor parameter rename from `chainId` to `chain` is small but breaking
* The `timestamp` is in **milliseconds** and replaces nonce as the uniqueness mechanism 
* `metadata` is a `bytes32` whose application-defined semantics are not detailed in the migration docs (**needs verification**).
* `builder` is a `bytes32` builder code that replaces V1's `POLY_BUILDER_*` HMAC headers 





## 2. Where builders and makers most commonly get stuck

The official migration guide is short, the operational pain points it does not enumerate are these.

Actually , most migration bugs are obvious, `initialize client failed`, `endpoint version`,`auth headers are wrong`, `SDK version`

*The expensive bugs are state bugs.*

Below is an flow migration dependency chain 

```
wallet
  -> signature type
    -> collateral token PUSD
      -> allowance
        -> order construction
          -> CLOB acceptance
            -> off-chain match
              -> on-chain settlement
                -> position update
                  -> reconciliation
```


Let me take some examples step by step  .

* **Wallet Signature type**

  `0=EOA`, `1=POLY_PROXY` (Magic/email), `2=POLY_GNOSIS_SAFE` (browser wallet auto-deploys this — most accounts)

  with `3` referenced in V2 quickstart troubleshooting and inferred to be EIP-1271 smart-contract wallet 

* **Collateral wrapping PUSD**

  API-only traders must wrap `USDC.e` into `PUSD` via the `CollateralOnramp.wrap()` function before any V2 order will settle, and the onramp can revert with `OnlyUnpaused()` if admin-paused.

* **Allowance scopes**

  An old USDC.e approval to the V1 exchange does not authorize the V2 onramp;

  An approval to the onramp does not authorize the V2 exchange to pull pUSD

* **Builder-code attribution drift** 

  if [`builderCode`](https://docs.polymarket.com/v2-migration#what%E2%80%99s-new) env var is empty or wrong, 

  your builder leaderboard will silently show zero and your revenue share will not accrue, with no error






## 3.Collateral and Funding Path: USDC.e, pUSD, Wrap / Unwrap

**V2 collateral is [PUSD](https://polygonscan.com/address/0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB).**

There are  three Polygon "USD" tokens must keep straight.

| Token  | Contract Address                           |                              |
| ------ | ------------------------------------------ | ---------------------------- |
| USDC.e | 0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174 | Polymarket V1 collateral     |
| USDC   | 0x3c499c542cEF5E3811e1192ce70d8cC03d5c3359 | new Version of circle issued |
| PUSD   | 0xC011a7E12a19f7B1f670d46F03B03f3342E82DFB | Polymarket V2 collateral.    |

```
USDC.e balance -> approval -> CollateralOnramp.wrap() 

-> pUSD balance 

-> trade
-> CollateralOfframp.unwrap()
```

If your bot still assumes USDC.e is the trading collateral after v2, you are likely wrong.



* **Why “not enough balance” can happen even when you have funds**

  By understanding our previous path, we will be able to identify exactly where the error from .

  (1) balance is in USDC.e, not pUSD — wrap first

  (2) pUSD allowance to the V2 Exchange (or Neg Risk Exchange) not enough 

  (3) wrapped funds into pUSD, but under the wrong address.

  (4) trading from a different signer / funder than expected.

  

* **`OnlyUnpaused()` reverts**

  The CollateralOnramp can be paused. 

  If a wrap call reverts with `OnlyUnpaused()` , the protocol is in a paused state — do not retry blindly; check operator status



## 4.**Wallet, Proxy Wallet, Signature Type, and Relayer**

V2 migration does not change old wallet types(`0` `1` `2`)

```rust
#[repr(u8)]
pub enum SignatureType {
    #[default]
    Eoa = 0,
    Proxy = 1,
    GnosisSafe = 2,
    /// EIP-1271 smart contract wallet signatures (V2 orders only)
    Poly1271 = 3,
}
```

**`0` = EOA** — signer holds funds. Typical for headless bots that fund their own EOA directly.

**`1` = POLY_PROXY** — Polymarket-custom proxy wallet, used for Magic/email-derived accounts.

**`2` = POLY_GNOSIS_SAFE** — modified Gnosis Safe, used for browser-wallet accounts. **This is the most common live account type today.** 

**`3`** — referenced in V2 quickstart troubleshooting ("0, 1, 2, or 3"). 

Most likely EIP-1271 smart-contract wallet given V2's broader smart-account support, but **the explicit mapping is not in the docs I could retrieve and is needs verification**.

```solidity
/// @notice See EIP-1271: https://eips.ethereum.org/EIPS/eip-1271
function isValidSignature(bytes32 _message, bytes memory _signature)
        public
        view
        virtual
        returns (bytes4 magicValue)
```



* **`timestamp` issue**

  V2 removes the`nonce` field from the order struct entirely, replacing it with a millisecond `timestamp` for uniqueness 

  the migration docs also describe a new operator-side `pauseUser`/`unpauseUser` mechanism that replaces on-chain cancel, and whether *that* mechanism is asynchronous in a way that still admits a settlement-failure window .

  * **TODO- to be validated after V2 launch**



* **Builder credentials vs relayer keys, in V2 terms**

​	V2 changes the Builder Program flow,  and V2 replaces the old builder authentication 	flow for order attribution with a native `builderCode` attached directly to each order.

​	**builderCode**: attribution / revenue sharing / per-order builder field

​	**relayer credentials**:gasless transaction / relayer authentication path

​	

The community OrderBookTrade [`rs-builder-relayer-client`](https://github.com/OrderBookTrade/rs-builder-relayer-client) README distinguishes a `builder` HMAC mode (gasless, not address-bound) from a `relayer_key` mode (address-bound, from `polymarket.com/settings`). 



For market makers, relayer credentials may matter more because execution and settlement reliability matter more than attribution.



## 5.**Order lifecycle: SUBMIT, MATCH, MINED, CONFIRMED, FAILED**

Per [https://docs.polymarket.com/market-data/websocket/user-channel], the documented states are:

| Status      | Terminal | Description                                                  |
| :---------- | :------- | :----------------------------------------------------------- |
| `MATCHED`   | No       | Trade has been matched and sent to the executor service by the operator |
| `MINED`     | No       | Trade observed to be mined into the chain, no finality threshold established |
| `CONFIRMED` | Yes      | Trade has achieved strong probabilistic finality and was successful |
| `RETRYING`  | No       | Trade transaction has failed (revert or reorg) and is being retried/resubmitted by the operator |
| `FAILED`    | Yes      | Trade has failed and is not being retried                    |

```
PENDING_SUBMIT
  └─ ACCEPTED ───┬─ OPEN ───┬─ PARTIALLY_FILLED
  └─ REJECTED   │           │
                │           ├─ CANCELED
                │           │
                │           └─ MATCHED ─── MINED ─── CONFIRMED
                │                            │
                │                            └─ ROLLED_BACK ─── FAILED
                │
                └─ EXPIRED

  Orthogonal: UNKNOWN, RECONCILE_NEEDED
```

`MATCHED -> MINED -> CONFIRMED `

the process is good !

But a production bot needs to handle the failure branches, and this is more important.



* **Why CLOB matching is fast but settlement is async**

  Matching is a memory operation against an order book on Polymarket's servers

  But on-chain settlement is a Polygon transaction subject to mempool, blockspace, gas, and contract revert risk

  A trade can show MATCHED within tens of milliseconds and reach CONFIRMED 2-3 seconds later — or reverted (this is the root cause of [ghost fill](https://github.com/OrderBookTrade/GhostGuard/blob/main/docs/ghost-fills-polymarket.md) ).

* **Whether to poll tx hash**

  WebSocket might drop, reorder, or arrive late, the reliable pattern is to  take the operator-reported tx hash from the MATCHED→MINED transition, poll Polygon RPC and read the `Trade` event emitted by the V2 exchange contract from the receipt logs.

  

## 6.Ghost fills: the risk that deserves the most respect

I have previously written an article about using [GhostGuard](https://github.com/OrderBookTrade/GhostGuard/blob/main/docs/ghost-fills-polymarket.md) to handle ghost fills.

> A **ghost fill** is a trade that appears matched and filled on the off-chain CLOB — your local state, the operator's UI, and even the WebSocket trade event all say it happened — 
>
> but on-chain settlement never lands successfully, and the position never changes. 
>
> The fill briefly shows in the book and your account, then disappears.



The exact v2 behavior must be tested after launch.


|Official v2 changes remove the old nonce model, which should address nonce-increment-specific failure modes, and add [100 block support](https://github.com/Polymarket/ctf-exchange-v2/blob/main/src/exchange/mixins/UserPausable.sol#L14-L38) for `UserPausable` 

But removing nonce does not automatically prove that every ghost-fill-like state inconsistency is impossible.



* **Does V2 fully solve ghost fills?** 

  I think probably it eliminates the specific `incrementNonce` exploit, because the order struct no longer contains a nonce. 

  The structural **off-chain-match / on-chain-settle gap remains,** and any new operator-side cancel mechanism (`pauseUser`/`unpauseUser`) introduces its own failure surface that cannot be evaluated until live. 

  Treat your detection and reconciliation logic as **necessary regardless of V2** — the cost of running it is low, and the cost of being wrong is large.

  

**TODO** definitive judgment is to be validated after V2 launch



## 7. WebSocket, Order State, and Reconciliation

**WebSocket is necessary.**

But only websocket is not sufficient .

**A robust bot should not use a single source of truth.**

* **REST polling + WS reconciliation**

  A ideal method is: take WS as primary, REST snapshot on reconnect and every 5–10 minutes during steady-state, REST snapshot every 30s during settlement windows for orders in MATCHED. 

  On-chain log poll every 30s for any order in (MATCHED, MINED, RETRYING).



* **Error Handling** 

  Whether a builder is profitable depends not only on their strategy but also on the stability of the service infrastructure.

  > * Ws messages can be delayed
  > * Ws messages can be duplicated
  > * REST and WS can temporarily disagree
  > * messages can arrive out of order
  > * API | Onchain state mismatch

  

  Yesh , tbh ,  make the bot system stable ,and handle all these error is a not easy , it is very heavy .

  

  **<u>Trading infra is heavy because money is heavy.</u>**





## 8. **Maker Launch Checklist**

#### 8.1 Pre-launch

Cancel all open V1 orders ahead of the maintenance window 

They will be wiped anyway, but explicit cancel lets you reconcile your local state cleanly, and avoids any edge case where the wipe surfaces as ROLLED_BACK in your stream.

The order book will be empty for a window post-resume regardless of whether matching is technically live.

#### 8.2 During maintenance 

<u>Trading is paused</u>

Do not poll the CLOB at high frequency during the pause — it both wastes effort and risks tripping rate limits when service comes back. 

Hold a watcher on a single status endpoint (or just on PolygonScan for V2 contract activity) to detect resume.



#### 8.3 After cutover

Keep size small, Increase reconciliation frequency.

First 24 hours: cap notional per order at 10–25% of normal limits, double reconciliation cadence (e.g., 15s instead of 30s), set the kill switch to trigger on lower thresholds (e.g., 3 ghost-fill candidates in 15 minutes instead of 10).



## 9.Latency, FIFO, matching engine, region



**Whether V2 matching latency improved is unknown until measured.** 





## 10.**From Zero to One: Practical Migration Path**

Each step has a purpose, a check, common errors, and a success criterion.

---

**Step 1 — Read official migration docs.**

***Purpose**:* establish ground truth, not third-party summaries.

***Check**:* you have read `docs.polymarket.com/v2-migration` `docs.polymarket.com/resources/contracts`, the V2 changelog entries, and the Help Center cutover article.
***Common errors**:* relying on community summaries (including this article) without reading the source. 

***Success**:* you can name the V2 exchange addresses, the EIP-712 domain version, and the cutover time without looking them up.

---

**Step 2 — Clone/install `clob-client-v2`.** 

***Purpose**:* get the V2 SDK in the dependency tree. 

***Check**:* `npm install @polymarket/clob-client-v2 viem` (or the Python equivalent `py-clob-client-v2`); verify `package.json` has the new dep and the old `@polymarket/clob-client` and `@polymarket/builder-signing-sdk` are removed. 

***Common errors**:* both packages installed simultaneously, importing from the V1 path by accident, version mismatch with viem. 

***Success**:* `import { ClobClient } from "@polymarket/clob-client-v2"` resolves, build passes lint.

---

**Step 3 — Prepare wallet/credentials.** 

***Purpose**:* know which key signs and which address holds funds. 

***Check**:* you have your EOA private key in a secrets manager, your funder address (proxy/Safe/EOA), your `signatureType` (0/1/2), and your existing API key triplet `{ key, secret, passphrase }`. 

***Common errors:*** using a fresh EOA that is not associated with the existing proxy/Safe and getting balance mismatches; forgetting that V1 keys carry over and re-deriving unnecessarily. 

***Success**:* L1 EIP-712 sign-in works against staging, L2 HMAC requests authenticate cleanly.

---

**Step 4 — Confirm collateral / contract address.**

 ***Purpose**:* avoid pasted-from-Discord traps.

 ***Check**:* fetch `docs.polymarket.com/resources/contracts` directly; record` V2 CTF Exchange, Neg Risk Exchange, pUSD, CollateralOnramp, CollateralOfframp` addresses; cross-verify each on PolygonScan with a "Polymarket" label. 

***Common errors:*** using V1 addresses, pasting an address with a single-character typo (the historical `0xaB45c54AB...` vs `0xaB45c5A4B...` is a documented example), trusting community cheatsheets without re-fetch. 

***Success**:* every address in your config matches the official contracts page, byte-for-byte.

---

**Step 5 — Check balance/allowance.**

 ***Purpose**:* ensure the funder address can actually trade. 

***Check**:* on-chain balance of USDC.e and pUSD on funder address; allowance of USDC.e to CollateralOnramp; allowance of pUSD to V2 CTF Exchange (and Neg Risk if applicable). 

***Common errors:*** allowance set to V1 contracts; allowance to onramp but not to exchange; pUSD held in EOA but funder pointed at proxy. 

***Success**:* `balanceOf(funder)` and `allowance(funder, exchange)` return expected non-zero values for the token you actually intend to trade in.

---

**Step 6 — Initialize client.**

 ***Purpose**:* get a fully-authenticated `ClobClient` against V2. 

***Check**:* construct against staging host first, then production after cutover; pass `chain: Chain.POLYGON`; pass `creds`; pass `signatureType` and `funderAddress` explicitly if not EOA. 

***Common errors:*** using `chainId` instead of `chain`; omitting `funderAddress` for proxy/Safe accounts; setting unsupported constructor params (`tickSizeTtlMs` is removed). 

***Success**:* `client.getOk()` (or any unauthenticated GET equivalent) returns 200; an authenticated GET against `/orders` returns your historical orders.

---

**Step 7 — Query markets / order book.** 

***Purpose**:* verify read-side wiring before write-side. 

***Check**:* `getMarkets`, `getOrderBook(tokenId)` for a known liquid market, and `getClobMarketInfo(conditionID)` (V2 new). 

***Common errors:*** token IDs mistakenly used as market IDs; tick-size mismatches when constructing test orders. 

***Success**:* book returns non-empty bids and asks at expected tick sizes; market info returns fee config.

---

**Step 8 — Place tiny test order.**

 ***Purpose**:* exercise the full signing → submission → ACK path. 

***Check**:* construct GTC limit at a non-crossing price, $1–$5 notional; observe ACK; observe order on book via book query.

***Common errors:*** invalid signature (wrong domain version, wrong verifyingContract, wrong signatureType); insufficient balance/allowance; tick-size violation. 

***Success**:* order accepted, returned `order_id`, visible on book.

---

**Step 9 — Subscribe WebSocket updates.**

 ***Purpose**:* validate the user channel is wired. 

***Check**:* connect to `wss://ws-subscriptions-clob.polymarket.com` (V1 URL; V2 unchanged per migration FAQ); subscribe to user channel for your address; observe heartbeat; place another tiny order and watch for ACCEPTED then OPEN events. 

***Common errors:*** wrong subscription topic, wrong auth on WS, dropped without reconnect logic.

 ***Success**:* every state transition you trigger via REST is reflected on the WS within seconds.

---

**Step 10 — Confirm order state.**

***Purpose**:* end-to-end state tracking.

***Check**:* your local state machine updates correctly from the WS events for ACCEPTED, OPEN, MATCHED (if filled), MINED, CONFIRMED, CANCELED. 

***Common errors:*** missing `RECONCILE_NEEDED` transitions; mutating state on optimistic events without a rollback path. 

***Success:*** a planned out-of-order event (replayed) does not corrupt local state.

---

**Step 11 — Test cancel.** 

***Purpose**:* the cancel path is asymmetric to the place path. 

***Check:*** place, observe OPEN, cancel, observe canceled state on WS, confirm absence on REST snapshot. 

***Common errors**:* cancel by stale client id; cancelling before ACCEPTED. 

***Success**:* every cancel converges, no orphan orders left after a stress run.

---

**Step 12 — Test fill.**

 ***Purpose**:* exercise the matching path. 

***Check**:* place an aggressive order that crosses, observe MATCHED. 

***Common errors:*** fee misaccounting (V2 fees are operator-set; do not pre-compute V1-style); wrong price math due to 6-decimal scaling on amounts (`$100 = 100_000_000`). 

***Success:*** MATCHED event arrives, fill recorded in local log with matching server timestamp.

---

**Step 13 — Test settlement / CONFIRMED.** 

***Purpose**:* confirm the on-chain leg works. 

***Check**:* observe MATCHED → MINED → CONFIRMED on WS; independently verify the Trade event in V2 CTF Exchange logs via Polygon RPC against the tx hash. 

***Common errors:*** assuming MATCHED is terminal; treating MINED as final; missing the on-chain check.

 ***Success**:* on-chain log matches WS event; position changes by the expected amount.

---

**Step 14 — Reconciliation.** 

***Purpose**:* catch ghost fills and dropped events. 

***Check**:* reconciler runs every 30s, compares local fill log + server fill log + on-chain log; flags discrepancies; resolves by REST snapshot + on-chain RPC.

 ***Common errors:*** reconciler not running; reconciler running but not actually flagging; reconciler flagging but bot ignoring `RECONCILE_NEEDED` orders.

 ***Success:*** a deliberate forced disagreement (e.g., delete a local fill) is detected and surfaced within one cycle.

---

**Step 15 — Small flow live.**

 ***Purpose**:* see how the system behaves under real adverse selection at small size. 

***Check:*** run for ~24 hours at 10–25% of normal size; track ghost-fill count, RETRYING count, FAILED count, latency distributions. 

***Common errors:*** scaling up before the dashboard is real; ignoring the first ghost-fill event as "noise." 

***Success:*** zero unresolved discrepancies; latency distributions stable; ghost-fill incidence within tolerance.

---

**Step 16 — Full cutover.** 

***Purpose:*** return to normal size after data supports it. 

***Check:*** you have at least 24 hours of clean small-flow data; you have a documented kill-switch threshold; you have on-call coverage. 

***Common errors:*** full size before reconciliation has been stress-tested; assuming V1 size limits port unchanged. 

***Success:*** normal operation, dashboards green, kill switch armed.

---

**Step 17 — Keep kill switch + rollback plan.** 

***Purpose:*** the migration is over but the operational risk isn't.

 ***Check:*** kill switch tested in production at low size; rollback plan documents how to disable strategies, withdraw collateral, unwind positions; runbook for the most likely incidents (ghost fill spike, RPC failure, WS disconnect, operator outage). 

***Common errors:*** shelving the kill switch once the migration "feels stable."

 ***Success:*** kill switch fires correctly during a planned drill; rollback plan executes without error.

## 
