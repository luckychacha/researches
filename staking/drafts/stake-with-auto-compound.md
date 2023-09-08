# Stake With Auto Compound

## Analyse

### Initialize the genesis state of a blockchain

>   1. Round Information Initialization: Sets up the information for the first round and stores it in the Round storage item.

```Rust

#[derive(Copy, Clone, PartialEq, Eq, Encode, Decode, RuntimeDebug, TypeInfo)]
/// The current round index and transition information
pub struct RoundInfo<BlockNumber> {
    /// Current round index
    pub current: RoundIndex,
    /// The first block of the current round
    pub first: BlockNumber,
    /// The length of the current round in number of blocks
    pub length: u32,
}

#[pallet::genesis_build]
pub struct GenesisConfig<T: Config> {
    // ...
    let round: RoundInfo<T::BlockNumber> =
            RoundInfo::new(1u32, 0u32.into(), self.blocks_per_round);
    <Round<T>>::put(round);
    // ...
}

impl<
		B: Copy + sp_std::ops::Add<Output = B> + sp_std::ops::Sub<Output = B> + From<u32> + PartialOrd,
	> RoundInfo<B>
{
	pub fn new(current: RoundIndex, first: B, length: u32) -> RoundInfo<B> {
		RoundInfo {
			current,
			first,
			length,
		}
	}
	/// Check if the round should be updated
	pub fn should_update(&self, now: B) -> bool {
		now - self.first >= self.length.into()
	}
	/// New round
	pub fn update(&mut self, now: B) {
		self.current = self.current.saturating_add(1u32);
		self.first = now;
	}
}

```

### `#[pallet::hooks]` on_initialize 

The `on_initialize` function serves as the entry point at the beginning of each block. This function performs a crucial check to determine whether it's time to initiate a new staking round. If the conditions are met, several key actions are executed:
   - Round Information Update: The metadata for the current round is updated to reflect the new round.
      - 判断是否需要更新 round，如果 `BlockNumber` - `round.first` > `round.length`
          - 需要更新 round
              - 更新 round
              - select_top_candidates
              - prepare_staking_payouts
          - 不需要更新 round
            - handle_delayed_payouts
```Rust
    #[pallet::hooks]
    impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
        fn on_initialize(n: T::BlockNumber) -> Weight {
            let mut weight = T::WeightInfo::base_on_initialize();

            let mut round = <Round<T>>::get();
            if round.should_update(n) {
                // mutate round
                round.update(n);
                // notify that new round begin
                weight = weight.saturating_add(T::OnNewRound::on_new_round(round.current));
                // pay all stakers for T::RewardPaymentDelay rounds ago
                weight = weight.saturating_add(Self::prepare_staking_payouts(round.current));
                // select top collator candidates for next round
                let (extra_weight, collator_count, _delegation_count, total_staked) =
                    Self::select_top_candidates(round.current);
                weight = weight.saturating_add(extra_weight);
                // start next round
                <Round<T>>::put(round);
                // snapshot total stake
                <Staked<T>>::insert(round.current, <Total<T>>::get());
                Self::deposit_event(Event::NewRound {
                    starting_block: round.first,
                    round: round.current,
                    selected_collators_number: collator_count,
                    total_balance: total_staked,
                });
                // account for Round and Staked writes
                weight = weight.saturating_add(T::DbWeight::get().reads_writes(0, 2));
            } else {
                weight = weight.saturating_add(Self::handle_delayed_payouts(round.current));
            }

            // add on_finalize weight
            //   read:  Author, Points, AwardedPts
            //   write: Points, AwardedPts
            weight = weight.saturating_add(T::DbWeight::get().reads_writes(3, 2));
            weight
        }
        // ....
    }

```

### `fn handle_delayed_payouts`

In cases where a new round is not initiated, the `handle_delayed_payouts` function is invoked. This function is responsible for:
  - RewardPaymentDelay is 2.
  - let delay = T::RewardPaymentDelay::get();
  - 轮次小于 delay 的，不需要 payout
  - 如果 DelayedPayouts 能查询到本轮到待支付信息，就执行 pay_one_collator_reward
    - 如果 pay_one_collator_reward 执行成功，返回的是 `RewardPayment::Finished`，移除 DelayedPayouts 和 Points 中的本轮到数据记录
  - 如果在 DelayedPayouts 没查询到本轮到待支付信息，就返回 `Weight::from_parts(0u64, 0)`
```Rust

    /// Wrapper around pay_one_collator_reward which handles the following logic:
    /// * whether or not a payout needs to be made
    /// * cleaning up when payouts are done
    /// * returns the weight consumed by pay_one_collator_reward if applicable
    fn handle_delayed_payouts(now: RoundIndex) -> Weight {
        let delay = T::RewardPaymentDelay::get();

        // don't underflow uint
        if now < delay {
            return Weight::from_parts(0u64, 0);
        }

        let paid_for_round = now.saturating_sub(delay);

        if let Some(payout_info) = <DelayedPayouts<T>>::get(paid_for_round) {
            let result = Self::pay_one_collator_reward(paid_for_round, payout_info);

            // clean up storage items that we no longer need
            if matches!(result.0, RewardPayment::Finished) {
                <DelayedPayouts<T>>::remove(paid_for_round);
                <Points<T>>::remove(paid_for_round);
            }
            result.1 // weight consumed by pay_one_collator_reward
        } else {
            Weight::from_parts(0u64, 0)
        }
    }

```

### `fn pay_one_collator_reward`

The `pay_one_collator_reward` function is specialized for handling the reward distribution to a single collator. It performs the following:
   - **Reward Calculation**: The function calculates the reward based on the staking ratio between the collator and the delegators.
   - **Payout Execution**: Once the reward is calculated, it is disbursed to the respective parties.
    - https://showme.redstarplugin.com/s/s:gkDHbEbs
```Rust

/// Payout a single collator from the given round.
///
/// Returns an optional tuple of (Collator's AccountId, total paid)
/// or None if there were no more payouts to be made for the round.
pub(crate) fn pay_one_collator_reward(
    paid_for_round: RoundIndex,
    payout_info: DelayedPayout<BalanceOf<T>>,
) -> (RewardPayment, Weight) {
    // 'early_weight' tracks weight used for reads/writes done early in this fn before its
    // early-exit codepaths.
    let mut early_weight = Weight::zero();

    // TODO: it would probably be optimal to roll Points into the DelayedPayouts storage
    // item so that we do fewer reads each block
    let total_points = <Points<T>>::get(paid_for_round);
    early_weight = early_weight.saturating_add(T::DbWeight::get().reads_writes(1, 0));

    if total_points.is_zero() {
        // TODO: this case is obnoxious... it's a value query, so it could mean one of two
        // different logic errors:
        // 1. we removed it before we should have
        // 2. we called pay_one_collator_reward when we were actually done with deferred
        //    payouts
        log::warn!("pay_one_collator_reward called with no <Points<T>> for the round!");
        return (RewardPayment::Finished, early_weight);
    }

    let collator_fee = payout_info.collator_commission;
    let collator_issuance = collator_fee * payout_info.round_issuance;
    if let Some((collator, state)) =
        <AtStake<T>>::iter_prefix(paid_for_round).drain().next()
    {
        // read and kill AtStake
        early_weight = early_weight.saturating_add(T::DbWeight::get().reads_writes(1, 1));

        // Take the awarded points for the collator
        let pts = <AwardedPts<T>>::take(paid_for_round, &collator);
        // read and kill AwardedPts
        early_weight = early_weight.saturating_add(T::DbWeight::get().reads_writes(1, 1));
        if pts == 0 {
            return (RewardPayment::Skipped, early_weight);
        }

        // 'extra_weight' tracks weight returned from fns that we delegate to which can't be
        // known ahead of time.
        let mut extra_weight = Weight::zero();
        let pct_due = Perbill::from_rational(pts, total_points);
        let total_paid = pct_due * payout_info.total_staking_reward;
        let mut amt_due = total_paid;

        let num_delegators = state.delegations.len();
        let mut num_paid_delegations = 0u32;
        let mut num_auto_compounding = 0u32;
        let num_scheduled_requests = <DelegationScheduledRequests<T>>::get(&collator).len();
        if state.delegations.is_empty() {
            // solo collator with no delegators
            extra_weight = extra_weight
                .saturating_add(T::PayoutCollatorReward::payout_collator_reward(
                    paid_for_round,
                    collator.clone(),
                    amt_due,
                ))
                .saturating_add(T::OnCollatorPayout::on_collator_payout(
                    paid_for_round,
                    collator.clone(),
                    amt_due,
                ));
        } else {
            // pay collator first; commission + due_portion
            let collator_pct = Perbill::from_rational(state.bond, state.total);
            let commission = pct_due * collator_issuance;
            amt_due = amt_due.saturating_sub(commission);
            let collator_reward = (collator_pct * amt_due).saturating_add(commission);
            extra_weight = extra_weight
                .saturating_add(T::PayoutCollatorReward::payout_collator_reward(
                    paid_for_round,
                    collator.clone(),
                    collator_reward,
                ))
                .saturating_add(T::OnCollatorPayout::on_collator_payout(
                    paid_for_round,
                    collator.clone(),
                    collator_reward,
                ));

            // pay delegators due portion
            for BondWithAutoCompound {
                owner,
                amount,
                auto_compound,
            } in state.delegations
            {
                let percent = Perbill::from_rational(amount, state.total);
                let due = percent * amt_due;
                if !due.is_zero() {
                    num_auto_compounding += if auto_compound.is_zero() { 0 } else { 1 };
                    num_paid_delegations += 1u32;
                    Self::mint_and_compound(
                        due,
                        auto_compound.clone(),
                        collator.clone(),
                        owner.clone(),
                    );
                }
            }
        }

        extra_weight =
            extra_weight.saturating_add(T::WeightInfo::pay_one_collator_reward_best(
                num_paid_delegations,
                num_auto_compounding,
                num_scheduled_requests as u32,
            ));

        (
            RewardPayment::Paid,
            T::WeightInfo::pay_one_collator_reward(num_delegators as u32)
                .saturating_add(extra_weight),
        )
    } else {
        // Note that we don't clean up storage here; it is cleaned up in
        // handle_delayed_payouts()
        (RewardPayment::Finished, Weight::from_parts(0u64, 0))
    }
}


```

### `fn mint_and_compound`

The `mint_and_compound` function is designed to not only payout rewards to delegators but also to compound a portion of these rewards for future growth. It operates as follows:
   - **Immediate Payout**: Initially, the rewards are paid out to the delegators.
   - **Compounding**: A portion of the rewards is then compounded based on a predefined ratio to facilitate growth.

```rust
    	/// Mint and compound delegation rewards. The function mints the amount towards the
		/// delegator and tries to compound a specified percent of it back towards the delegation.
		/// If a scheduled delegation revoke exists, then the amount is only minted, and nothing is
		/// compounded. Emits the [Compounded] event.
		pub fn mint_and_compound(
			amt: BalanceOf<T>,
			compound_percent: Percent,
			candidate: T::AccountId,
			delegator: T::AccountId,
		) {
			if let Ok(amount_transferred) =
				T::Currency::deposit_into_existing(&delegator, amt.clone())
			{
				Self::deposit_event(Event::Rewarded {
					account: delegator.clone(),
					rewards: amount_transferred.peek(),
				});

				let compound_amount = compound_percent.mul_ceil(amount_transferred.peek());
				if compound_amount.is_zero() {
					return;
				}

				if let Err(err) = Self::delegation_bond_more_without_event(
					delegator.clone(),
					candidate.clone(),
					compound_amount.clone(),
				) {
					log::debug!(
						"skipped compounding staking reward towards candidate '{:?}' for delegator '{:?}': {:?}",
						candidate,
						delegator,
						err
					);
					return;
				};

				Pallet::<T>::deposit_event(Event::Compounded {
					delegator,
					candidate,
					amount: compound_amount.clone(),
				});
			};
		}
	}
```

### Delegator Staking Enhancement

> The `delegation_bond_more_without_event` function is tailored to assist delegators in increasing their staked amount. It also enables the option for automatic compounding. The function performs these tasks:
>    - **Pending Undelegation Check**: It first checks if there are any pending undelegation requests.
>    - **Stake Increase**: If no undelegation requests are found, the function increases the delegator's staked amount accordingly.

```Rust
    /// This function exists as a helper to delegator_bond_more & auto_compound functionality.
    /// Any changes to this function must align with both user-initiated bond increases and
    /// auto-compounding bond increases.
    /// Any feature-specific preconditions should be validated before this function is invoked.
    /// Any feature-specific events must be emitted after this function is invoked.
    pub fn delegation_bond_more_without_event(
        delegator: T::AccountId,
        candidate: T::AccountId,
        more: BalanceOf<T>,
    ) -> Result<
        (bool, Weight),
        DispatchErrorWithPostInfo<frame_support::dispatch::PostDispatchInfo>,
    > {
        let mut state = <DelegatorState<T>>::get(&delegator).ok_or(Error::<T>::DelegatorDNE)?;
        ensure!(
            !Self::delegation_request_revoke_exists(&candidate, &delegator),
            Error::<T>::PendingDelegationRevoke
        );

        let actual_weight = T::WeightInfo::delegator_bond_more(
            <DelegationScheduledRequests<T>>::get(&candidate).len() as u32,
        );
        let in_top = state
            .increase_delegation::<T>(candidate.clone(), more)
            .map_err(|err| DispatchErrorWithPostInfo {
                post_info: Some(actual_weight).into(),
                error: err,
            })?;

        Ok((in_top, actual_weight))
    }
```


### `#[pallet::hooks]` on_finalize 

> 用于给 `BlockAuthor` 增加积分

```Rust
#[pallet::hooks]
impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
    fn on_finalize(_n: T::BlockNumber) {
        Self::award_points_to_block_author();
    }
}

fn award_points_to_block_author() {
    let author = T::BlockAuthor::get();
    let now = <Round<T>>::get().current;
    let score_plus_20 = <AwardedPts<T>>::get(now, &author).saturating_add(20);
    <AwardedPts<T>>::insert(now, author, score_plus_20);
    <Points<T>>::mutate(now, |x| *x = x.saturating_add(20));
}

```