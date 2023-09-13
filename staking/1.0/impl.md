# Implement


## architecture

### hook on pallet-session
`pallet-session` hooks, `rotate_session` will be called when a block is initialized.

```Rust

#[pallet::hooks]
impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
    /// Called when a block is initialized. Will rotate session if it is the last
    /// block of the current session.
    fn on_initialize(n: T::BlockNumber) -> Weight {
        if T::ShouldEndSession::should_end_session(n) {
            Self::rotate_session();
            T::BlockWeights::get().max_block
        } else {
            // NOTE: the non-database part of the weight for `should_end_session(n)` is
            // included as weight for empty block, the database part is expected to be in
            // cache.
            Weight::zero()
        }
    }
}

```

### logic in `rotate_session`

```Rust
// Increment session index.
let session_index = session_index + 1;
<CurrentIndex<T>>::put(session_index);

T::SessionManager::start_session(session_index);

// Get next validator set.
let maybe_next_validators = T::SessionManager::new_session(session_index + 1);
let (next_validators, next_identities_changed) =
    if let Some(validators) = maybe_next_validators {
        // NOTE: as per the documentation on `OnSessionEnding`, we consider
        // the validator set as having changed even if the validators are the
        // same as before, as underlying economic conditions may have changed.
        (validators, true)
    } else {
        (Validators::<T>::get(), false)
    };
```


### our own logic

```Rust

#[pallet::config]
pub trait Config:
    frame_system::Config
    + pallet_staking::Config
    + pallet_balances::Config
    + pallet_session::Config
{
    type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
    type RemainderDestination: OnUnbalanced<NegativeImbalance<Self>>;
    type PrivilegedOrigin: EnsureOrigin<Self::RuntimeOrigin>;
    type WrappedSessionManager: pallet_session::SessionManager<
        <Self as pallet_session::Config>::ValidatorId,
    >;
    type TimeProvider: UnixTime;
    type CurrencyInfo: CurrencyInfo;
}

```

### staking genesis_build default reward destination is Staked

```Rust
#[pallet::genesis_build]
impl<T: Config> GenesisBuild<T> for GenesisConfig<T> {
    fn build(&self) {
        for &(ref stash, ref controller, balance, ref status) in &self.stakers {
            frame_support::assert_ok!(<Pallet<T>>::bond(
                T::RuntimeOrigin::from(Some(stash.clone()).into()),
                T::Lookup::unlookup(controller.clone()),
                balance,
                RewardDestination::Staked,
            ));
        }
    }
}

```



--------------

let election_result: BoundedVec<_, MaxWinnersOf<T>> = if is_genesis {
    let result = <T::GenesisElectionProvider>::elect().map_err(|e| {
        log!(warn, "genesis election provider failed due to {:?}", e);
        Self::deposit_event(Event::StakingElectionFailed);
    });

    result
        .ok()?
        .into_inner()
        .try_into()
        // both bounds checked in integrity test to be equal
        .defensive_unwrap_or_default()
} else {
    let result = <T::ElectionProvider>::elect().map_err(|e| {
        log!(warn, "election provider failed due to {:?}", e);
        Self::deposit_event(Event::StakingElectionFailed);
    });
    result.ok()?
};
let exposures = Self::collect_exposures(election_result);

collect_exposures -> exposures

trigger_new_era(start_session_index, exposures)
|-Self::store_stakers_info(exposures, new_planned_era)
    |-<ErasStakersClipped<T>>::insert(&new_planned_era, &stash, exposure_clipped);


we calculate reward each session is:

let exposure = <pallet_staking::ErasStakersClipped<T>>::get(era, &ledger.stash);
let validator_exposure_part = Perbill::from_rational(exposure.own, exposure.total);
let validator_staking_payout = validator_exposure_part * validator_leftover_payout;

we do the payout is update Ledger:
fn make_payout(stash: &T::AccountId, amount: BalanceOf<T>) -> Option<PositiveImbalanceOf<T>> {
    match dest {
    RewardDestination::Staked => {
        pallet_staking::Pallet::<T>::bonded(stash)
            .and_then(|c| pallet_staking::Pallet::<T>::ledger(&c).map(|l| (c, l)))
            .and_then(|(controller, mut l)| {
                l.active += amount;
                l.total += amount;
                let r = T::Currency::deposit_into_existing(stash, amount).ok();
                T::Currency::set_lock(
                    STAKING_ID,
                    &l.stash,
                    l.active,
                    WithdrawReasons::all(),
                );
                Ledger::insert(&controller, l);
                r
            })
        }
    }
}