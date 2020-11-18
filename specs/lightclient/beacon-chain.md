# Ethereum 2.0 Light Client Support

## Table of contents

<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Introduction](#introduction)
- [Custom types](#custom-types)
- [Constants](#constants)
- [Configuration](#configuration)
  - [Misc](#misc)
  - [Time parameters](#time-parameters)
  - [Domain types](#domain-types)
- [Containers](#containers)
  - [Extended containers](#extended-containers)
    - [`BeaconBlockBody`](#beaconblockbody)
    - [`BeaconState`](#beaconstate)
  - [New containers](#new-containers)
    - [`SyncCommittee`](#synccommittee)
- [Helper functions](#helper-functions)
  - [Misc](#misc-1)
  - [Beacon state accessors](#beacon-state-accessors)
    - [`get_sync_committee_indices`](#get_sync_committee_indices)
    - [`get_sync_committee`](#get_sync_committee)
  - [Block processing](#block-processing)
    - [Sync committee processing](#sync-committee-processing)
  - [Epoch processing](#epoch-processing)
    - [Final updates](#final-updates)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

## Introduction

This is a standalone beacon chain patch adding light client support via sync committees.

## Custom types

| Name | SSZ equivalent | Description |
| - | - | - |

## Constants

| Name | Value |
| - | - | 
| `BASE_REWARDS_PER_EPOCH` | `uint64(5)` |

## Configuration

### Misc

| Name | Value |
| - | - | 
| `MAX_SYNC_COMMITTEE_SIZE` | `uint64(2**8)` (= 256) |

### Time parameters

| Name | Value | Unit | Duration |
| - | - | :-: | :-: |
| `EPOCHS_PER_SYNC_COMMITTEE_PERIOD` | `Epoch(2**8)` (= 256) | epochs | ~27 hours |

### Domain types

| Name | Value |
| - | - |
| `DOMAIN_SYNC_COMMITTEE` | `DomainType('0x07000000')` |

## Containers

### Extended containers

#### `BeaconBlockBody`

```python
class BeaconBlockBody(phase0.BeaconBlockBody):
    # Light client
    sync_committee_bits: Bitlist[MAX_SYNC_COMMITTEE_SIZE]
    sync_committee_signature: BLSSignature
```

#### `BeaconState`

```python
class BeaconState(phase0.BeaconState):
    # Light client
    current_sync_committee: SyncCommittee
    next_sync_committee: SyncCommittee
```

### New containers

#### `SyncCommittee`

```python
class SyncCommittee(Container):
    pubkeys: List[BLSPubkey, MAX_SYNC_COMMITTEE_SIZE]
    pubkeys_aggregate: BLSPubkey
```

## Helper functions

### Misc

### Beacon state accessors

#### `get_sync_committee_indices`

```python
def get_sync_committee_indices(state: BeaconState, epoch: Epoch) -> Sequence[ValidatorIndex]:
    """
    Return the sync committee indices for a given state and epoch.
    """
    base_epoch = Epoch((max(epoch // EPOCHS_PER_SYNC_COMMITTEE_PERIOD, 1) - 1) * EPOCHS_PER_SYNC_COMMITTEE_PERIOD)
    active_validator_count = uint64(len(get_active_validator_indices(state, base_epoch)))
    sync_committee_size = min(active_validator_count, MAX_SYNC_COMMITTEE_SIZE)
    seed = get_seed(state, base_epoch, DOMAIN_SYNC_COMMITTEE)
    i, output = 0, []
    while i < active_validator_count and len(output) < sync_committee_size:
        candidate_index = indices[compute_shuffled_index(uint64(i), active_validator_count, seed)]
        random_byte = hash(seed + uint_to_bytes(uint64(i // 32)))[i % 32]
        effective_balance = state.validators[candidate_index].effective_balance
        if effective_balance * MAX_RANDOM_BYTE >= MAX_EFFECTIVE_BALANCE * random_byte:
            output.append(candidate_index)
        i += 1
    return output
```

#### `get_sync_committee`

```python
def get_sync_committee(state: BeaconState, epoch: Epoch) -> SyncCommittee:
    """
    Return the sync committee for a given state and epoch.
    """
    indices = get_sync_committee_indices(state, epoch)
    validators = [state.validators[index] for index in indices]
    pubkeys = [validator.pubkey for validator in validators]
    return SyncCommittee(pubkeys, bls.AggregatePKs(pubkeys))
```

### Block processing

```python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    phase0.process_block(state, block)
    process_sync_committee(state, block.body)
```

#### Sync committee processing

```python
def process_sync_committee(state: BeaconState, body: BeaconBlockBody) -> None:
    # Verify sync committee bitfield length
    committee_indices = get_sync_committee_indices(state, get_current_epoch(state))
    assert len(body.sync_committee_bits) == len(committee_indices)

    # Verify sync committee aggregate signature signing over the previous slot block root
    previous_slot = max(state.slot, Slot(1)) - Slot(1)
    participant_indices = [committee_indices[i] for i in range(len(committee_indices)) if body.sync_committee_bits[i]]
    participant_pubkeys = [state.validators[participant_index].pubkey for participant_index in participant_indices]
    domain = get_domain(state, DOMAIN_SYNC_COMMITTEE, compute_epoch_at_slot(previous_slot))
    signing_root = compute_signing_root(get_block_root_at_slot(state, previous_slot), domain)
    assert bls.FastAggregateVerify(participant_pubkeys, signing_root, body.sync_committee_signature)

    # Reward sync committee participants
    participant_rewards = Gwei(0)
    active_validator_count = uint64(len(get_active_validator_indices(state, get_current_epoch(state))))
    for participant_index in participant_indices:
        base_reward = get_base_reward(state, participant_index)
        reward = Gwei(base_reward * active_validator_count // len(committee_indices) // SLOTS_PER_EPOCH)
        increase_balance(state, participant_index, reward)
        participant_rewards += reward

    # Reward beacon proposer
    increase_balance(state, get_beacon_proposer_index(state), Gwei(participant_rewards // PROPOSER_REWARD_QUOTIENT))
```

### Epoch processing

#### Final updates

```python
def process_final_updates(state: BeaconState) -> None:
    phase0.process_final_updates(state)
    next_epoch = get_current_epoch(state) + Epoch(1)
    if next_epoch % EPOCHS_PER_SYNC_COMMITTEE_PERIOD == 0:
        state.current_sync_committee = state.next_sync_committee
        state.next_sync_committee = get_sync_committee(state, next_epoch + EPOCHS_PER_SYNC_COMMITTEE_PERIOD)
```