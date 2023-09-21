### what's different between active_era and current_era

ActiveEra is the era that we are on.
CurrentEra is the era that staking is on.

For example, if the election failed. ActiveEra will still be increased, but the CurrentEra not. And one day you found they are not equal, which means your chain's election stalled. ActiveEra - CurrentEra eras is the stalled time.

```

start_session -> start_era -> increase ActiveEra
new_session -> try_trigger_new_era -> trigger_new_era -> increase CurrentEra

```