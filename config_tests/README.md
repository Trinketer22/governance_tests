# Config contract testing project

## Ignored messages

- Not masterchain
- Bounced
- No body

## Response message format

Unless specified otherwise is following:

``` func
config_resp#_ 
  answer:uint32
  query_id:uint64
```

## Next validators set

Happens when new validator set is proposed by elector.

### Reject cases

- Source address must be elector contract address
- Proposed vset should not be expired: `(valid_since > now()) & (valid_until > valid_since)`

### On reject

No change in contract state

- Message is sent back to the requesting address.
- Message answer is `op::response::update_vset_reject`
- Message mode is **64**.

### On success

Config parameter `config_params::next_validators_set` is set to vset specified

- Message is sent back to the requesting  address.
- Message answer is `op::response:;update_vset_confirm`
- Message mode is **64**.

## New vote proposal

Happens when someone proposes new vote for config parameters or updates existing proposal.

### Relevant get methods

- `proposal_storage_price` Calculates proposal storage price.
- `get_proposal` get's proposal by it's hash.
- `list_proposals` get's list of all proposals.

### Reject cases

- Config param current value hash in proposal should match actual value hash (`error::proposal::old_mismatch`)
- Mandatory parameter can't be nullified (proposal hash < 0 and parameter id in mandatory). (`error::proposal::mandatory_nullified`)
- Param value depth is limited to < 128. (`error:proposal::too_deep`)
- If critical config parameter is in question, then critical flag should be set in proposal. (`error::proposal::critical_flag_missing`)
- Proposal `expire_in` field should >= minimal storage time for config parameter category (ordinary or critical) (`error::proposal::expired`)
- Message value should exceed storage price by at least 1 TON. (`error::proposal::insufficient_fee`).

On update of existing proposal there are some extra reject cases:

- Status update from critical to non critical is not possible. (`error::proposal::critical_flag_mismatch`)
- New `expire_in` should be >= than one in previous proposal state. (`error::proposal::already_exists`)
- Message value shoud exceed recalculated (if needed) storage price by at least 1 TON. (`error::proposal::insufficient_fee`).

### On reject

No change in contract state

- Message is sent to the requesting address.
- Message answers is set accordingly to the reject case error value.
- Message mode is **64**.

### On success

- Proposal created/updated according to parameters specified.
- Contract reserves storage price on the balance.
- Message is sent back to requesting address.
- Message answer is `op::response:;proposal_accepted`.
- Message mode is **128**.

## Vote for config proposal

Happens when vote is submitted for proposal.

### Relevant get methods

- `proposal_storage_price` Calculates proposal storage price.
- `get_proposal` get's proposal by it's hash.
- `list_proposals` get's list of all proposals.

### Reject cases

- Signature tag should match:`tag::signature_challenge`. (throws `error::proposal_vote::tag_mismatch`)
- Signature should match validator public key. (throws `error::proposal_vote::sig_mismatch`)
- Proposal with hash specified should exist. (status -1)
- Proposal in questions should not be expired. (status -1)
- Proposal exists only limited number of rounds (status -1)
- Proposal lost too many rounds (status -1)
- Only one vote per validator is accepted. (status -2)

### On reject

If vote is rejected due to the fact that it wasn't accepted for too many rounds,  
proposal in question is removed.  
Otherwise no canges to the contract state.

- Message is sent back to the requesting address.
- Message answer is set to `op::response::vote_result` + status specified in reject case
- Message mode is **64**.

### On success

Proposal wins/losses and round count adjusted accordingly.

- Proposal remaining weight decreases.
- If remaining weight < 0, proposal wins a round ( 6 - critical flag)
- Else remaining weight decreases. (2)
- If minimum required rounds won, and config parameter is set to value proposed.
- Message is sent back to the requesting address.
- Message answer is set to `op::response::vote_result` + status specified in success case.
- Message mode is **64**.

## Vote for config proposal from elector contract

**NEW** *config* and *elector* contracts **ONLY**!

Happens when vote for proposal is done via elector contract.  
Pretty much same deal as before, but elector authorizes validator by address instead of signature.

### Relevant get methods

- `proposal_storage_price` Calculates proposal storage price.
- `get_proposal` get's proposal by it's hash.
- `list_proposals` get's list of all proposals.

### Reject cases

- Message should originate from elector address. (`error::unauthorized`)
- Voting validators set should be current. (`error::expired_vset`)

Rest are identical to [Vote for config proposal](#vote-for-config-proposal) case.

- Proposal with hash specified should exist. (status -1)
- Proposal in questions should not be expired. (status -1)
- Proposal exists only limited number of rounds (status -1)
- Proposal lost too many rounds (status -1)
- Only one vote per validator is accepted. (status -2)

### On reject

Identical to [Vote for config proposal](#vote-for-config-proposal) case.  

If vote is rejected due to the fact that it wasn't accepted for too many rounds,  
proposal in question is removed.
Otherwise no changes to the contract state.

- Message is sent back to the requesting address.
- Message answer is set to `op::response::vote_result` + status specified in reject case
- Message mode is **64**.

### On success

Identical to [Vote for config proposal](#vote-for-config-proposal) case.  

- Proposal remaining weight decreases.
- If remaining weight < 0, proposal wins a round ( 6 - critical flag)
- Else remaining weight decreases. (2)
- If minimum required rounds won, and config parameter is set to value proposed.
- Message is sent back to the requesting address.
- Message answer is set to `op::response::vote_result` + status specified in success case.
- Message mode is **64**.

## External message checks

Certain functionality of config contract is accessible via external messages.

Each external message should satisfy following criteria:

- Message should have valid timestamp (not expired).
- Message depth should be <= 128.
- Message seqno should match contract seqno counter field (has 2^32 - 1 period).
- Message signature should be valid. (Checked against stored public keys).

## Vote for config proposal via external message

Another way of passing config parameter change vote.
Identical to the previous one, except general external message checks above.

## Other config actions

Other actions also performed via external messages.
There are two modes available:

- Master key.
- Multi key (**NEW CONTRACT ONLY**).

### Master key

Pretty much self explanatory. Single message signed by master key
is enough to force any action below.

### Multi key

2/3 of public keys have to vote for specific action to be performed.

### Set config value

*OP*:`op::update_config_parameter`  
Sets specified config index to a specified value.

### Code upgrade

*OP*: `op::op::set_new_code`  
Allows to update config contract code.  
Produces *set code* action.

### Change config master key

*OP*: `op::update_config_key`  
Sets new master key.

### Change elector code

Sends message to *elector contract*:

- Message is sent to *elector contract* address.
- Message value is ~ 1 TON (1 >> 30) in old contract and strictly 1 in new one.
- Message *op* is `op::set_new_code`.
- Message *query id* equals current timestamp.

## Update current validators set

Check happens on every ticktock event.

### Reject cases

- Next validators set (*Config param* **36**) should be defined(not *null*).
- Validators set *valid since* field should be >= current time.

### On reject

Nothing happens.

### On success

- Current validators set (*Config param* **34**) is set to the  Next validators set (*Config param* **36**) value.
- Next validators set to **null** value.

## Update proposal round status

Check happens on every ticktock event if no vset update happened that ticktock.

### Reject cases

- No proposals present

### On reject

Nothing happens
