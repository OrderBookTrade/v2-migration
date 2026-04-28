# Polymarket CLOB v2 Migration Guide



If your bot manages real capital, the Polymarket CLOB v2 migration is not a documentation task.

It is a risk-control task.



Polymarket CLOB v2 changes more than the client package. 



[The official migration guide](https://docs.polymarket.com/v2-migration) describes this as a coordinated upgrade across new Exchange contracts, a rewritten CLOB backend, and a new collateral token, pUSD. 

The cutover is scheduled for April 28, 2026 around 11:00 UTC, with roughly one hour of downtime, and all open orders are expected to be wiped during the cutover.  



**That means** builders, makers, and API traders need to re-check the full trading path:

```
wallet -> signature -> collateral -> allowance -> order -> match -> settlement -> position
```









## TL;DR





You don't want to lose money ,right ?

This is a comprehensive guide for migrating from Polymarket CLOB v1 to Polymarket CLOB v2.





## **1. What CLOB v2 Actually Changes**



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



## 11. Post-V2 Launch Verification Plan



## 12. Conclusion
