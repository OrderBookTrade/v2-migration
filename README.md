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









## 2. Where builders and makers most commonly get stuck



## 3.Collateral and Funding Path: USDC.e, pUSD, Wrap / Unwrap



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
