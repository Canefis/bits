# BIT-0014: Proportional burn

- **BIT Number:** 0014
- **Title:** Proportional Burn
- **Author(s):** Greg Zaitsev
- **Discussions-to:** [URL for discussion thread]
- **Status:** Draft
- **Type:** Core
- **Created:** 2025-07-25
- **Updated:** 2025-07-25

**Note:** This BIT in fact tables the [PR #1883](https://github.com/opentensor/subtensor/pull/1883).

## 🔍 Abstract

This BIT proposes burning alpha and root TAO for subnet owner and all subnet validators proportionally to alpha burned in burn keys (all hotkeys associated with owner's coldkey).

## 🔧 Motivation

Subnet owners utilize the alpha burn keys a lot in favor of receiving cut and validation emissions, which reduces quality of subnet and bittensor in general.

## 🧪 Specification

- Burn incentives of all hotkeys that are owned by subnet owner's coldkey and SubnetOwnerHotkey.
- Calculate burned proportion of owner's miner incentive
- Burn the same proportion of owner cut and validator emission

### Implementation

Happens entirely in `distribute_dividends_and_incentives` function.

```rust
pub fn distribute_dividends_and_incentives(
    netuid: NetUid,
    owner_cut: AlphaCurrency,
    incentives: BTreeMap<T::AccountId, AlphaCurrency>,
    alpha_dividends: BTreeMap<T::AccountId, U96F32>,
    tao_dividends: BTreeMap<T::AccountId, U96F32>,
) {
    // Distribute mining incentives and calculate the proportion of burned incentive
    let mut total_incentive = U64F64::saturating_from_num(0);
    let mut burned_incentive = U64F64::saturating_from_num(0);
    for (hotkey, incentive) in incentives {
        log::debug!("incentives: hotkey: {:?}", incentive);

        total_incentive = total_incentive.saturating_add(U64F64::saturating_from_num(incentive));

        // Burn miner emission for all hotkeys associated with subnet owner coldkey
        // Also, calculate the burned proportion
        let mut skip_and_burn = false;
        if let Ok(owner_hotkey) = SubnetOwnerHotkey::<T>::try_get(netuid) {
            if hotkey == owner_hotkey {
                skip_and_burn = true;
            }
        }
        if Owner::<T>::get(&hotkey) == SubnetOwner::<T>::get(netuid) {
            skip_and_burn = true;
        }
        if skip_and_burn {
            burned_incentive = burned_incentive.saturating_add(U64F64::saturating_from_num(incentive));
            log::debug!(
                "incentives: hotkey: {:?} is SN owner hotkey, skipping {:?}",
                hotkey,
                incentive
            );
            continue;
        }

        // Increase stake for miner.
        Self::increase_stake_for_hotkey_and_coldkey_on_subnet(
            &hotkey.clone(),
            &Owner::<T>::get(hotkey.clone()),
            netuid,
            incentive,
        );
    }
    let remaining_proportion = total_incentive.saturating_sub(burned_incentive).safe_div(total_incentive);

    // Distribute the owner cut.
    if let Ok(owner_coldkey) = SubnetOwner::<T>::try_get(netuid) {
        if let Ok(owner_hotkey) = SubnetOwnerHotkey::<T>::try_get(netuid) {
            let owner_cut_after_burn = AlphaCurrency::from(remaining_proportion.saturating_mul(U64F64::saturating_from_num(owner_cut)).saturating_to_num::<u64>());

            // Increase stake for owner hotkey and coldkey.
            log::debug!(
                "owner_hotkey: {:?} owner_coldkey: {:?}, owner_cut: {:?}",
                owner_hotkey,
                owner_coldkey,
                owner_cut_after_burn
            );
            let real_owner_cut = Self::increase_stake_for_hotkey_and_coldkey_on_subnet(
                &owner_hotkey,
                &owner_coldkey,
                netuid,
                owner_cut_after_burn,
            );
            // If the subnet is leased, notify the lease logic that owner cut has been distributed.
            if let Some(lease_id) = SubnetUidToLeaseId::<T>::get(netuid) {
                Self::distribute_leased_network_dividends(lease_id, real_owner_cut);
            }
        }
    }

    // Distribute alpha divs.
    let _ = AlphaDividendsPerSubnet::<T>::clear_prefix(netuid, u32::MAX, None);
    for (hotkey, alpha_divs) in alpha_dividends {
        let mut alpha_divs_after_burn = U96F32::saturating_from_num(remaining_proportion).saturating_mul(alpha_divs);

        // Get take prop
        let alpha_take: U96F32 =
            Self::get_hotkey_take_float(&hotkey).saturating_mul(alpha_divs_after_burn);
        // Remove take prop from alpha_divs
        alpha_divs_after_burn = alpha_divs_after_burn.saturating_sub(alpha_take);
        // Give the validator their take.
        log::debug!("hotkey: {:?} alpha_take: {:?}", hotkey, alpha_take);
        Self::increase_stake_for_hotkey_and_coldkey_on_subnet(
            &hotkey,
            &Owner::<T>::get(&hotkey),
            netuid,
            tou64!(alpha_take).into(),
        );
        // Give all other nominators.
        log::debug!("hotkey: {:?} alpha_divs: {:?}", hotkey, alpha_divs_after_burn);
        Self::increase_stake_for_hotkey_on_subnet(&hotkey, netuid, tou64!(alpha_divs_after_burn).into());
        // Record dividends for this hotkey.
        AlphaDividendsPerSubnet::<T>::mutate(netuid, &hotkey, |divs| {
            *divs = divs.saturating_add(tou64!(alpha_divs_after_burn).into());
        });
        // Record total hotkey alpha based on which this value of AlphaDividendsPerSubnet
        // was calculated
        let total_hotkey_alpha = TotalHotkeyAlpha::<T>::get(&hotkey, netuid);
        TotalHotkeyAlphaLastEpoch::<T>::insert(hotkey, netuid, total_hotkey_alpha);
    }

    // Distribute root tao divs.
    let _ = TaoDividendsPerSubnet::<T>::clear_prefix(netuid, u32::MAX, None);
    for (hotkey, root_tao) in tao_dividends {
        let mut root_divs_after_burn = U96F32::saturating_from_num(remaining_proportion).saturating_mul(root_tao);

        // Get take prop
        let tao_take: U96F32 = Self::get_hotkey_take_float(&hotkey).saturating_mul(root_divs_after_burn);
        // Remove take prop from root_tao
        root_divs_after_burn = root_divs_after_burn.saturating_sub(tao_take);
        // Give the validator their take.
        log::debug!("hotkey: {:?} tao_take: {:?}", hotkey, tao_take);
        let validator_stake = Self::increase_stake_for_hotkey_and_coldkey_on_subnet(
            &hotkey,
            &Owner::<T>::get(hotkey.clone()),
            NetUid::ROOT,
            tou64!(tao_take).into(),
        );
        // Give rest to nominators.
        log::debug!("hotkey: {:?} root_tao: {:?}", hotkey, root_divs_after_burn);
        Self::increase_stake_for_hotkey_on_subnet(
            &hotkey,
            NetUid::ROOT,
            tou64!(root_divs_after_burn).into(),
        );
        // Record root dividends for this validator on this subnet.
        TaoDividendsPerSubnet::<T>::mutate(netuid, hotkey.clone(), |divs| {
            *divs = divs.saturating_add(tou64!(root_divs_after_burn));
        });
        // Update the total TAO on the subnet with root tao dividends.
        SubnetTAO::<T>::mutate(NetUid::ROOT, |total| {
            *total = total
                .saturating_add(validator_stake.to_u64())
                .saturating_add(tou64!(root_divs_after_burn));
        });
    }
}
```

## 📘 Reference Implementation

The feature was once implemented and tabled:
https://github.com/opentensor/subtensor/pull/1883

## Test Plan

```rust
// cargo test --package pallet-subtensor --lib -- tests::coinbase::test_incentive_to_subnet_owner_is_burned --exact --show-output
#[test]
fn test_incentive_to_subnet_owner_is_burned() {
    new_test_ext(1).execute_with(|| {
        let subnet_owner_ck = U256::from(0);
        let subnet_owner_hk = U256::from(1);

        let other_ck = U256::from(2);
        let other_hk = U256::from(3);
        Owner::<Test>::insert(other_hk.clone(), other_ck.clone());

        let netuid = add_dynamic_network(&subnet_owner_hk, &subnet_owner_ck);

        let pending_tao: u64 = 1_000_000_000;
        let pending_alpha = AlphaCurrency::ZERO; // None to valis
        let owner_cut = AlphaCurrency::ZERO;
        let mut incentives: BTreeMap<U256, AlphaCurrency> = BTreeMap::new();

        // Give incentive to other_hk
        incentives.insert(other_hk, 10_000_000.into());

        // Give incentives to subnet_owner_hk
        incentives.insert(subnet_owner_hk, 10_000_000.into());

        // Verify stake before
        let subnet_owner_stake_before =
            SubtensorModule::get_stake_for_hotkey_on_subnet(&subnet_owner_hk, netuid);
        assert_eq!(subnet_owner_stake_before, 0.into());
        let other_stake_before = SubtensorModule::get_stake_for_hotkey_on_subnet(&other_hk, netuid);
        assert_eq!(other_stake_before, 0.into());

        // Distribute dividends and incentives
        SubtensorModule::distribute_dividends_and_incentives(
            netuid,
            owner_cut,
            incentives,
            BTreeMap::new(),
            BTreeMap::new(),
        );

        // Verify stake after
        let subnet_owner_stake_after =
            SubtensorModule::get_stake_for_hotkey_on_subnet(&subnet_owner_hk, netuid);
        assert_eq!(subnet_owner_stake_after, 0.into());
        let other_stake_after = SubtensorModule::get_stake_for_hotkey_on_subnet(&other_hk, netuid);
        assert!(other_stake_after > 0.into());
    });
}

// cargo test --package pallet-subtensor --lib -- tests::coinbase::test_incentive_to_subnet_owner_owned_hotkey_is_burned --exact --show-output
#[test]
fn test_incentive_to_subnet_owner_owned_hotkey_is_burned() {
    new_test_ext(1).execute_with(|| {
        let subnet_owner_ck = U256::from(0);
        let subnet_owner_hk = U256::from(1);

        let other_hk = U256::from(3);
        Owner::<Test>::insert(other_hk.clone(), subnet_owner_ck.clone());

        let netuid = add_dynamic_network(&subnet_owner_hk, &subnet_owner_ck);

        let pending_tao: u64 = 1_000_000_000;
        let pending_alpha = AlphaCurrency::ZERO; // None to valis
        let owner_cut = AlphaCurrency::ZERO;
        let mut incentives: BTreeMap<U256, AlphaCurrency> = BTreeMap::new();

        // Give incentive to other_hk
        incentives.insert(other_hk, 10_000_000.into());

        // Give incentives to subnet_owner_hk
        incentives.insert(subnet_owner_hk, 10_000_000.into());

        // Verify stake before
        let subnet_owner_stake_before =
            SubtensorModule::get_stake_for_hotkey_on_subnet(&subnet_owner_hk, netuid);
        assert_eq!(subnet_owner_stake_before, 0.into());
        let other_stake_before = SubtensorModule::get_stake_for_hotkey_on_subnet(&other_hk, netuid);
        assert_eq!(other_stake_before, 0.into());

        // Distribute dividends and incentives
        SubtensorModule::distribute_dividends_and_incentives(
            netuid,
            owner_cut,
            incentives,
            BTreeMap::new(),
            BTreeMap::new(),
        );

        // Verify stake after
        let subnet_owner_stake_after =
            SubtensorModule::get_stake_for_hotkey_on_subnet(&subnet_owner_hk, netuid);
        assert_eq!(subnet_owner_stake_after, 0.into());
        let other_stake_after = SubtensorModule::get_stake_for_hotkey_on_subnet(&other_hk, netuid);
        assert_eq!(other_stake_after, 0.into());
    });
}

// cargo test --package pallet-subtensor --lib -- tests::coinbase::test_owner_cut_burn_proportional_to_incentive --exact --show-output
#[test]
fn test_owner_cut_burn_proportional_to_incentive() {
    new_test_ext(1).execute_with(|| {
        let subnet_owner_ck = U256::from(0);
        let subnet_owner_hk = U256::from(1);

        let other_ck = U256::from(2);
        let other_hk = U256::from(3);
        Owner::<Test>::insert(other_hk.clone(), other_ck.clone());

        let netuid = add_dynamic_network(&subnet_owner_hk, &subnet_owner_ck);

        let pending_tao: u64 = 1_000_000_000;
        let pending_alpha = AlphaCurrency::ZERO; // None to valis
        let owner_cut = AlphaCurrency::from(1_000_000_000);
        let mut incentives: BTreeMap<U256, AlphaCurrency> = BTreeMap::new();

        // Give incentive to other_hk
        incentives.insert(other_hk, 10_000_000.into());

        // Give incentives to subnet_owner_hk, total is 50/50 split
        incentives.insert(subnet_owner_hk, 10_000_000.into());
        let expected_owner_cut = owner_cut / AlphaCurrency::from(2);

        // Verify stake before
        let subnet_owner_stake_before =
            SubtensorModule::get_stake_for_hotkey_on_subnet(&subnet_owner_hk, netuid);
        assert_eq!(subnet_owner_stake_before, 0.into());
        let other_stake_before = SubtensorModule::get_stake_for_hotkey_on_subnet(&other_hk, netuid);
        assert_eq!(other_stake_before, 0.into());

        // Distribute dividends and incentives
        SubtensorModule::distribute_dividends_and_incentives(
            netuid,
            owner_cut,
            incentives,
            BTreeMap::new(),
            BTreeMap::new(),
        );

        // Verify stake after
        let subnet_owner_stake_after =
            SubtensorModule::get_stake_for_hotkey_on_subnet(&subnet_owner_hk, netuid);
        assert_eq!(subnet_owner_stake_after, expected_owner_cut);
        let other_stake_after = SubtensorModule::get_stake_for_hotkey_on_subnet(&other_hk, netuid);
        assert!(other_stake_after > 0.into());
    });
}
```

## © Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
