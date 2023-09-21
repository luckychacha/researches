### What's different between active_era and current_era?

ActiveEra is the era that we are on.
CurrentEra is the era that staking is on.

For example, if the election failed. ActiveEra will still be increased, but the CurrentEra not. And one day you found they are not equal, which means your chain's election stalled. ActiveEra - CurrentEra eras is the stalled time.

```

start_session -> start_era -> increase ActiveEra
new_session -> try_trigger_new_era -> trigger_new_era -> increase CurrentEra

```

#### Staking progress

[staking_progress](https://github.com/paritytech/substrate-api-sidecar/blob/8906816/src/controllers/pallets/PalletsStakingProgressController.ts#L61L75)

Note about 'active' vs. 'current' era: The _active_ era is the era currently being rewarded.
That is, an elected validator set will be in place for an entire active era, as long as none
are kicked out due to slashing. Elections take place at the end of each _current_ era, which
is the latest planned era, and may not equal the active era. Normally, the current era index
increments one session before the active era, in order to perform the election and queue the
validator set for the next active era. For example:

```

Time: --------->
CurrentEra:            1              |              2              |
ActiveEra:   |              1              |              2              |
SessionIdx:  |  1 |  2 |  3 |  4 |  5 |  6 |  7 |  8 |  9 | 10 | 11 | 12 | 13 | 14 |
Elections:                           ^                             ^
Set Changes:                               ^                             ^

```

