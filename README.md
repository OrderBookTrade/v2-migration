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

> * **Proxy wallets (EIP1271 Support )** 
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



## 2. Where builders and makers most commonly get stuck



## 3**.Collateral and Funding Path: USDC.e, pUSD, Wrap / Unwrap**



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
