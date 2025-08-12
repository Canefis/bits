# BIT-0016: Subnet Deregistration

- **BIT Number:** 0016
- **Title:** Subnet Deregistration
- **Author(s):** John Spiigot, Greg Zaitsev
- **Discussions-to:** [URL for discussion thread]
- **Status:** Draft
- **Type:** Core
- **Created:** 2025-08-12
- **Updated:** 2025-08-12

## 🔍 Abstract

This BIT proposes reenabling of subnet deregistration.

## 🔧 Motivation

Since the dTao launch, the non-functional subnets keep consuming emissions, as well as chain footprint and compute resources. The existing subnet deregistration (dissolve_network / remove_network) is no longer working correctly and is disabled.

## 🧪 Specification

The final specification is yet to be discussed. We can only propose several criteria to determine which subnets should be deregistered first. Note that because deregistration is an on-chain process, the criteria should only use data available on chain.

- Lack of demand for subnet Alpha token
- Low subnet price and emissions
- Centralized distribution of alpha / miner IP / miner coldkeys
- Unfavorable ratio and/or values of subnet parameters such as TAO and Alpha pool liquidity, Alpha circulating, Alpha issued, Alpha burned, FDV, Alpha volume, etc.

The current in-progress implementation may be used as a basis for such discussion, which is described below.

### Subnet Pruning
- Triggered when `SubnetLimit` is reached
- **Step 1:** Exclude subnets still within `NetworkImmunityPeriod`  
- **Step 2:** Among the rest, find the subnet with the lowest current emission  
- **Step 3:** If multiple share the same emission, pick the one with the earliest registration timestamp  

### Network Dissolution 

- In `dissolve_network` / `remove_network`, perform full dTao cleanup:
  - Return the registration cost to the owner (or owners in case if it was a crowdloan)
  - Destroy all α-in and α-out stakes  
  - Distribute remaining Tao to α-out stakers pro-rata
  - **Adjust the owner’s returned lock cost** by subtracting the portion of total emissions the owner actually received (`owner_received_emission = E * get_float_subnet_owner_cut()`), so the final refund is `max(0, lock_cost - owner_received_emission)`.
- Maintain **root-only** access to direct calls for now

### Explicit Subnet Limit
Add new sudo hyperparameter `SubnetLimit` starting at `256`.

### High-Level Flow

```text
New Registration → check slot cap?
  ├─ No → register network, grant immunity
  └─ Yes → prune one subnet → deregister → register new
Deregistration (manual or pruning) → destroy α-in/out → distribute Tao → remove network
```

## © Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
