# Elector testing plan

## Response messages format

Response messages have following format, unless explicitly specified otherwise.

```
elector_resp#_ 
  op:uint32
  query_id:uint64
  status:uint32
```

**Status** is either reject reason/operation or success status.  
Every default formatted message should be tested to:
-   *op* returned is expected.
-   *query_id* matches request *query_id*.
-   *status* is case specific.

## Simple transfer

Happens whenever someone sends message with empty body or *op* equal to **1** to the contract.

If there is no currently active validator set, message funds
are added to so called *nobody account* that will be later
distributed to validator bonuses.

In case there is active validator set, funds are added to
current set bonuses.


## New Stake

Happens when validator attempts to participate in elections and creates/adds it's stake.


### Relevant get methods:

-   `participates_in` checks whatever pub key presents in elections members dict.
-   `participant_list` gets lists of the participants among with their stake values.
-   `participant_list_extended` get list of the participants with all stake and elections meta data.

### Reject cases

-   No elections. **(reason:0)**
-   Source address should be from masterchain. **(reason:0)**
-   Elections already finished. **(reason:0)**
-   Message signature should be valid (checked with applicant provided pub key. (reason:1)
-   *msg_value* is lower than 1/4096 of total accumulated stake. (reason:2)
-   *msg_value* is lower than 1/4096 of total accumulated stake + confirmation fee ( in case *query_id* > 0 ). (reason:2)
-   Stake should be submitted for current elections ( *stake_at* should match *elect_at* ) (reason:3)
-   Stake can be made with one pub key from one address only. (reason:4)
-   Stake should be >= *min_stake*. (reason:5)
-   Stake max factor (*max_factor*) should be >= 1. (reason:6)


### On reject

On stake rejection no election parameters should change.  

-   Message is sent back to requesting address.
-   Message op equal to `op::response::stake_rejected`.
-   Message *reason* equal to reject reason.
-   Message mode should be **64**.

### On success

-   Total stake should increase by stake amount
-   Public key and stake data should be present in *members* dict of the elections.
-   If stake was already accepted from this pub key previously, it should sum up.
-   If request `query_id > 0` response message with value of 1 unit `op == op::response::stake_accepted` and *status* 0.

## Recover stake

Happens when validator wants to pull back unused/unfrozen part of it's stake.

### Relevant get methods:

-   `compute_returned_stake` checks if source address has and excess stake to be returned.

### Reject cases

-   Requesting source address should be from masterchain.
-   Some excess stake should be credited for requesting address.

### On reject

-   No data in contract state should change.
-   Message should be sent back to requesting address.
-   Message *op* equal to `op::not_allowed`. 
-   Message mode should be **64**


### On success

-   Requesting address credit record should be removed from credits dict -> *compute_returned_stake* should return **0**.
-   Message sent to the requesting address with attached value of previously credited stake.
-   Message *op* should equal to `op::response::stake_recover`.
-   Message mode should be **64**.

## Code upgrade

Happens when config contract initiates elector code update.

### Reject cases

-   *Config contract* address should be defined.
-   Requesting address should equal to config contract address.
-   Requesting address should be from masterchain.

### On reject

No data in contract state should change.

-   Message sent back to requesting address.
-   Message *op* equal to `op::not_supported`.
-   Message *status* equal to `op::set_new_code`.

### On success

-   Contract code should change.
-   Message sent back to *config contract*.
-   Message *op* should equal to `op::response::new_code_set`.
-   Message *status* should equal to `op::set_new_code`.
-   Message mode should be **64**.



## Validator elections announce

Happens on *ticktock* event if no elections were announced.  

### Relevant get methods:

-   `active_election_id` returns current election id.
-   `past_elections` returns past elections data.

### Reject

-   Current contract address should be an elector according to *config contract* data.
-   Current contract address should be on masterchain.
-   Current validator set should exist.
-   Current validator set should have correct format.
-   Current validator set should expire. (Otherwise why elect?)
-   Next validator set should not be present at the moment. (That's the elections result)
-   Election with current timestamp should not exist in past elections.

### On reject

Announce postponed till next time.

### On success

New elections should be set.

## Validator elections

Happens on *ticktock* event if elections were announced.

### Relevant get methods:

-   `active_election_id` returns current election id.
-   `past_elections` returns past elections data.
-   `participates_in` checks whatever pub key presents in elections members dict.
-   `participant_list` gets lists of the participants among with their stake values.
-   `participant_list_extended` get list of the participants with all stake and elections meta data.

### Reject cases

-   Elections are already ongoing.
-   No *config contract* to send result to.
-   Total number of participants is lower than minimum required. `min_validators`
-   Total submitted stake is less than minimum required. `min_total_stake`
-   Total true  stake (total stake with regard to factor) is lower than minimum required. `min_total_stake`
-   All of possible true stake has to be used during elections. ( throws exception )
-   Elections have finished.
-   Elections have failed. ( no validator satisfied election criteria)

### On reject

Elections are postponed till next announce.

### On success

-   All of the used stake should be frozen.
-   All of the unused stake should be credited to the validators according to their submission addresses.
-   Elections info stored in  past elections list.
-   Message is sent to the *config contract*.
-   Message *op* equal to `op::set_next_validator_set`.
-   Message *query_id* equal to the elections timestamp:`elect_at`.
-   Message mode should be **1**.

Message body references resulting validator set cell in the following format:

```
  validator_set#_
    validators_ext#12
    utime_since:uint32
    utime_until:uint32
    total:(## 16)
    main:(## 16)
    total_weight:uint64
    list:(HashmapE 16 ValidatorDescr)

```


## Update active validator set id

Happens on *ticktock* event.

### Relevant get methods:

-   `active_election_id` returns current election id.
-   `past_elections` returns past elections data.

### Reject cases

-   *Config contract* current validator set should change.
-   Active elections (if any) validator set should be equal to the *config contract* validator set.

### On reject

Wait till next *ticktock*.

### On success

-   If update happens during active elections, funds freezing time should be set to current time plus default funds freezing time.
-   Elections active id should change either to 0 or new elections id.
-   1/8 of the total *nobody balance* should be added to bonuses for the elections.



## Validator set installed

Happens on *ticktock* event if there are active elections. 
Checks if *config contract* installed new validator set.

### Relevant get methods:

-   `active_election_id` returns current election id.
-   `past_elections` returns past elections data.
-   `Config contract` current validator set or next validator set should equal elected one.

### Reject cases

-   Elections should be finished.
-   Elections data should be stored in `past_elections` dict. (We have active finished elecitons?)
-   *Config contract* current validator set or next validator set should be equal to active validator set.


### On reject

Postpone check till next time.

### On success

-   Current election should be deactivated.
-   [Update active validator set id](#update-active-validator-set-id) id triggered.

## Validators funds unfreeze

Happens on *ticktock* event.  
Loops through validator sets and checks if it's not active set and *unfreeze_at* time has come.  
If so, funds get credited under each validator submission address.



## Validators set accepted

Happens when *config contract* notifies elector
 that new validator set was accepted.  

### Relevant get methods:

-   `past_elections` returns past elections data.

### Reject cases

-   *Config contract* address should be defined.
-   Requesting address should equal to config contract address.
-   Requesting address should be from masterchain.
-   Current elections timestamp *elect_at* should equal to message *query_id*.
-   There should be active elections,
-   Elections should be in finished state at the moment of msg processing.

### On reject

No message is sent back and no data in contract state should change.

### On success

Elections have suceeded, Life goes on.  
No messages sent.



## Validators set rejected

Happens when *config contract* rejects validators set, proposed by *elector*.  

### Relevant get methods:

-   `compute_returned_stake` checks if source address has and excess stake to be retunred.
-   `past_elections` returns past elections data.
-   `participant_list_extended` get list of the participants with all stake and elections meta data.

### Reject cases

-   *Config contract* address should be defined.
-   Requesting address should equal to config contract address.
-   Requesting address should be from masterchain.
-   Current elections timestamp *elect_at* should equal to message *query_id*.
-   There should be active elections,
-   Elections should be in finished state at the moment of message processing.

### On reject

No message is sent back and no data in contract state should change.

### On success

-   Elections should be canceled (deactivated). 
-   All stakes should be credited back to validators.
-   If there are any bonus coins left, those should be credited too using following distribution formula: `total_bonuses * validator_stake / total_stake`.



## New complaint

Happens when someone wants to register complaint on validator behaviour.  

### Relevant get methods:
-   `complaint_storage_price` calculates storage price for complaint. 
-   `get_past_complaints` returns info on past elections complaints.
-   `list_complaints` returns complaints on specified elections.
-   `show_complaint`  returns complaint info by elections id and complaint cell hash.

### Reject cases

-   Requesting address should be from masterchain. (reason 1)
-   Elections with specified id should exists. (reason 2)
-   Complaint cell should not be deeper than 128 depth. (What's up with that?) (reason 3)
-   Complaint already expired. (reason 4)
-   Message value should be greater than `complaint_storage_price + ONE_UNIT`. (reason 5)
-   Validator in question should exists. (reason 6)
-   Validator stake should be larger than suggested fine. (reason 7)
-   Fine should be larger than price paid for complaint processing. (reason 8)
-   Duplicate complaints are not accepted. (reason 9)

### On reject

Now here reason value is added to *op* and not *status*. (Why?)  

-   Message should be sent back to requesting address
-   *Op* should be equal to `op::response::new_complaint + reason`
-   *Status* should be equal to `op::new_complaint`.
-   Message mode should be **64**.
-   No contract state data should change.

### On success

-   Complaint should be registered ( retrievable via get methods).
-   Contract should reserve storage fees on balance.
-   Message should be sent back to requesting address.
-   Message *op* should be equal to `op::response::new_complaint`.
-   Message *status* should be equal to `op::new_complaint`.
-   Message mode should be **128**.

## Vote for complaint

Happens when validators vote on complaint.

### Relevant get methods:

-   `get_past_complaints` returns info on past elections complaints.
-   `list_complaints` returns complaints on specified elections.


### Reject cases

-   Requesting address should be on masterchain. (throws `error::validator_params_mismatch`) **NEW_ONLY**.
-   Vote message body should have correct tag:*tag::complaint*. (throws `37`)
-   Vote message signature should be valid. (throws 34)
-   Validator should have been a participant of those elections by it's pub key. (throws `error::validator_params_mismatch`) **NEW_ONLY**
-   Validator should have been a participant of those elections by same src address. (throws `error::validator_params_mismatch`) **NEW_ONLY**
-   Complaint with hash in question should exist. (reason -1)
-   Elections with id in question should exist. (reason -2)
-   Vote directed to not current vset that already has >= 2/3 of votes (reset count otherwise)
-   Every validator can only vote once. (reason 0)

### On reject

Unless exception thrown:  
-   Message is sent back to requesting address.
-   Message *op* should equal `to op::vote_result + reason`.
-   Message *status* should equal to *op::vote_for_complaint*.
-   Message mode should be **64**.

Contract state should stay intact on reject.  

### On success
-   Voter should be added to complaint.
-   Remaining weight should change accordingly to voter's stake weight.
-   Message is sent back to requesting address.
-   Message *op* should equal to `op::vote_result + 1` if vote was accepted.
-   Message *op* should equal to `op::vote_result + 2` if vote was accepted and 2/3 majority acquired. 
-   Message *status* should equal to `op::vote_for_complaint`.
-   Message mode should be **64**.

## Vote for config changes

Happens when validators submit votes on config changes.  

### Reject cases

-   Requesting address should be on masterchain. (throws `error::validator_params_mismatch`)
-   Elections with id in question should exist. (throws  `error::no_elected_set`)
-   Validator should be a participant of those elections by it's pub key. (throws `error::validator_params_mismatch`)
-   Validator should be a participant of those elections by same src address. (throws `error::validator_params_mismatch`)
-   Public key supplied in vote message should match validator public key in vote (throws `error::validator_params_mismatch`)

### On reject

Exception is thrown and contract state should stay intact.

### On success

```
vote_payload_#
validator_idx:uint16
proposal_hash:uint256
active_elections_hash:uint256
```

#### Config notification

-   Message is sent to config contract.
-   Message *op* should equal to `op::vote_for_config_proposal_from_elector`.
-   Message *query_id* should equal to the request *query_id*.
-   Message mode should be **64**.
-   Message body should reference *vote_payload* cell.

#### Vote response
-   Message is sent back to requesting address.
-   *resp_msg* *op* should equal to `op::response::vote_for_config_proposal`.
-   *resp_msg* *status* should equal 0.
-   *resp_msg* mode should be **64**.
