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

Do not treat CLOB v2 migration as “update the SDK and restart the bot.”



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





## 4.**Wallet, Proxy Wallet, Signature Type, and Relayer**



## 5.**Order lifecycle: SUBMIT, MATCH, MINED, CONFIRMED, FAILED**



## 6.Ghost fills: the risk that deserves the most respect



## 7. WebSocket, Order State, and Reconciliation



## 8. **Maker Launch Checklist**

* Before 
* During
* After



## 9.Latency, FIFO, matching engine, region



## 10.**From Zero to One: Practical Migration Path**



## 11.Post-V2 Launch Verification Plan



## 12.Conclusion
