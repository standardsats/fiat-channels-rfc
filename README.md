## What is it

Fiat channels is a derived concept from original [Hosted channels](https://github.com/standardsats/hosted-channels-rfc), or HC, proposal by Anton Kumaigoroski. Channels of this type were initially implemented in Bitcoin Lightning Wallet and since then continued in new Simple Bitcoin Wallet and [IMMORTAN](https://github.com/btcontract/IMMORTAN) library.

Notable differences to the original [Hosted channels RFC](https://github.com/standardsats/hosted-channels-rfc):

* It adds special field into channel state for tracking last exchange rate. Thus, amount of satoshis in the channel and the last rate corresponding to last cross-signed state represent constant _fiat-denominated_ value.

* Fiat channels leverage resizing feature of the HC which helps to tackle sharp downward volatility of Bitcoin exchange rate and preserve purchasing power of the channel user. 

For the sake of convenience original RFC is mostly preserved.

## Specifics

* Hosted channel is always private and can never be closed in normal channel sense, its data may only be removed from Client wallet if there is no desire to use it anymore. Once removed, it can always be fully restored later provided that Client still has a wallet seed.

* Since hosted channels have no on-chain basis their `shortChannelId`s can not be derived from blockchain, as such they must be random and follow procedure defined in [#681](https://github.com/lightningnetwork/lightning-rfc/pull/681).

* Hosted channel ID is deterministic and is always known in advance for every Client/Host pair, it is derived as `sha256(bip69_lexicographical_order(Client node ID, Host node ID))`. This implies that each Client/Host pair may only have one hosted channel between them.

* A notion of _blockday_ is introduced as `current blockchain height / 144` to be used in messages specific to hosted channels, the intention is to ground them in real blockchain timeline.

## Invoking

* The peer feature flag for the Fiat Hosted Channel is `52972`, we also imply that the channel is resizeable (feature flag `52974`).

* Hosted channel must be explicitly invoked by Client on reconnection which is different from normal channel reestablish procedure, the rationale is that Client may have normal channels with Host and may not wish to use a hosted one. Once invoked, Host should reply with either `init_hosted_channel` message if no channel exists yet or with `last_cross_signed_state` if it does.

        +-------+                                           +-------+
        |       |--(1)------ invoke_hosted_channel -------->|       |
        |       |<-(2)------ init_hosted_channel -----------|       |
        |       |--(3)------ state_update ----------------->|       |
        |       |<-(4)------ state_update ------------------|       | // new channel established
        |       |                  or                       |       |
        |   A   |<-(2)------ last_cross_signed_state -------|   B   |
        |       |--(3)------ last_cross_signed_state ------>|       | // existing channel invoked
        +-------+                                           +-------+

        - where node A is Client and node B is Host

### The `invoke_hosted_channel` Message

1. type: 75535 (`invoke_hosted_channel`)
2. data:
  * [`chain_hash`:`chain_hash`]
  * [`u16`:`len`]
  * [`len*byte`:`refund_scriptpubkey`]
  * [`u16`:`len`]
  * [`len*byte`:`secret`]
  
#### Rationale

* By sending this message Client prompts Host to reveal last cross signed channel state or offer a new hosted channel if none exists for a given Client yet.

* `refund_scriptpubkey` is similar to `scriptpubkey` in [02-peer-protocol#closing-initiation-shutdown](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#closing-initiation-shutdown) and must be provided by Client for a case when Host would want to stop a hosted channel and refund the rest of the balance while Client is offline.

* `secret` is an optional data which can be used by Host to tweak channel parameters (non-zero initial Client balance, larger capacity, only allow Clients with secrets etc).

### The `init_hosted_channel` Message

1. type: 75533 (`init_hosted_channel`)
2. data:
  * [`u64`:`max_htlc_value_in_flight_msat`]
  * [`u64`:`htlc_minimum_msat`]
  * [`u16`:`max_accepted_htlcs`]
  * [`u64`:`channel_capacity_msat`]
  * [`u16`:`liability_deadline_blockdays`]
  * [`u64`:`minimal_onchain_refund_amount_satoshis`]
  * [`u64`:`initial_client_balance_msat`]
  * [`u64`:`initial_rate`]
  * [`u16`:`len`]
  * [`len*byte`:`features`]

#### Rationale

* This message is sent by Host in reply to `invoke_hosted_channel` if no hosted channel exists for a given Client yet.

* `liability_deadline_blockdays` specifies a period in blockdays after last `state_update` exchange during which the Host is definitely going to maintain a channel. That is, if there are no payments during `liability_deadline_blockdays` period then Host owes nothing to Client anymore. For example: if last exchanged `state_update` had `blockday` set to 1000 and `liability_deadline_blockdays` is set to 2000, then Host may not maintain a channel after blockday 3000 if it was not used all this time.

* `minimal_onchain_refund_amount_satoshis` specifies a minimal balance that Client must have in a hosted channel for a Host to consider refunding it on-chain using Client's `refund_scriptpubkey`.

* `features` here work similarly to node `features` field in [`Init`](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#the-init-message) message. While establishing a new hosted channel a node must drop it if unknown even features are present.

### The `last_cross_signed_state` Message

1. type: 75531 (`last_cross_signed_state`)
2. data:
  * [`u16`:`len`]
  * [`len*byte`:`last_refund_scriptpubkey`]
  * [[`init_hosted_channel`](#the-init_hosted_channel-message):`init_hosted_channel`]
  * [`u32`:`block_day`]
  * [`u64`:`local_balance_msat`]
  * [`u64`:`remote_balance_msat`]
  * [`u64`:`rate`] 
  * [`u32`:`local_updates`]
  * [`u32`:`remote_updates`]
  * [`u16`:`num_incoming_htlcs`]
  * [`num_incoming_htlcs`*[`update_add_htlc`](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#adding-an-htlc-update_add_htlc):`incoming_htlcs`]
  * [`u16`:`num_outgoing_htlcs`]
  * [`num_outgoing_htlcs`*[`update_add_htlc`](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#adding-an-htlc-update_add_htlc):`outgoing_htlcs`]
  * [`signature`:`remote_sig_of_local`]
  * [`signature`:`local_sig_of_remote`]

#### Rationale

* This message is sent by Host in reply to `invoke_hosted_channel` if it already exists for a given Client, once sent Host expects a similar `last_cross_signed_state` message from Client.

* Both parties must make sure that `remote_sig_of_local` and `local_sig_of_remote` signatures are valid, and that inverted `local_updates`/`remote_updates` numbers from remote `last_cross_signed_state` are the same as `local_updates`/`remote_updates` from local `last_cross_signed_state`.

* If local signature of remote `last_cross_signed_state` is valid but inverted `local_updates`/`remote_updates` numbers from remote `last_cross_signed_state` are higher than `local_updates`/`remote_updates` from local `last_cross_signed_state` then this means that local peer has fallen behind (or completely lost a state). Once this happens local peer must act as follows:
  * First, it must check if remote `last_cross_signed_state` points to one of future channel states which can be re-created locally. To enable this both peers store each local and remote `update_add_htlc`, `update_fulfill_htlc`, `update_fail_htlc`, `update_fail_malformed_htlc` message they send or receive in historic order until new `last_cross_signed_state` is reached. This vector of updates must be traversed with respected local `last_cross_signed_state` message created on each step and then compared against remote `last_cross_signed_state` update numbers. Once a match is found local peer applies it as current state, resolves all updates preceding this state and re-transmits all local updates following this state.
  * Second, if no matching future local `last_cross_signed_state` could be found then this means that local peer has completely lost it, in this case local peer should invert a remote `last_cross_signed_state` and apply it as its current state, then resolve all incoming HTLCs contained in remote `last_cross_signed_state`.
  
* Local state signature is obtained by creating a sigHash from localy generated `last_cross_signed_state`: `sha256(refund_scriptpubkey + liability_deadline_blockdays + minimal_onchain_refund_amount_satoshis + channel_capacity_msat + initial_client_balance_msat + block_day + local_balance_msat + remote_balance_msat + local_updates + remote_updates + incoming_htlcs + outgoing_htlcs)` and then signing it with `nodeId` private key.

## Normal operation

* First a peer sends `update_add_htlc`/`update_fulfill_htlc`/`update_fail_htlc`/`update_fail_malformed_htlc` messages which are then followed by local `state_update` message, which is then followed by remote `state_update`. Once both valid `state_update`s are collected a new cross-signed state is reached which in turn allows peers to resolve previous updates.

        +-------+                                            +-------+
        |       |----------- update_add_htlc #1 ------------>|       |
        |       |----------- state_update (is_terminal=0) -->|       | // A: here's my state update, now you have a new cross-signed state
        |       |<---------- state_update (is_terminal=1) ---|       | // B: got your update, anything else incoming?
        |       |----------- state_update (is_terminal=1) -->|       | // A: no other updates for now
        |       |<---------- state_update (is_terminal=1) ---|       | // B: OK, resolving
        |   A   |<---------- update_fulfill_htlc #1 ---------|   B   | 
        |       |<---------- state_update (is_terminal=0) ---|       |
        |       |----------- state_update (is_terminal=1) -->|       |
        |       |<---------- state_update (is_terminal=1) ---|       |
        |       |----------- state_update (is_terminal=1) -->|       |
        +-------+                                            +-------+

        Where B is Host and A is Client
        
        A more involved case with entwined updates, a diagram takes into account that it takes time for messages to get through, but message order is always preserved

        +-------+                                                   +-------+
        | 1, 0  |----------> update_add_htlc #A1                    | 0, 0  |
        | 1, 0  |----------> state_update #A1 (is_terminal=0)       | 0, 0  | // A inverts local state and signs B's `0, 1`, by not making it terminal A instructs B to not resolve an HTLC even if signature is correct but send an acking `state_update` in return
        | 1, 0  |            update_add_htlc #B1 <------------------| 1, 0  |
        | 1, 0  |            state_update #B1 (is_terminal=0) <-----| 1, 0  | // B inverts local state and signs A's `0, 1`, by not making it terminal B instructs A to not resolve an HTLC even if signature is correct but send an acking `state_update` in return
        | 1, 0  |                                                   | 1, 0  |
        | 1, 0  |            update_add_htlc #A1 ------------------>| 1, 1  |
        | 1, 0  |            state_update #A1 (is_terminal=0) ----->| 1, 1  | // A's `state_update` signs B's `1, 0` state, but B has `1, 1` by the time it arrives
        | 1, 1  |<---------- update_add_htlc #B1                    | 1, 1  |
        | 1, 1  |<---------- state_update #B1 (is_terminal=0)       | 1, 1  | // B's `state_update` signs A's `1, 0` state, but A has `1, 1` by the time it arrives
        | 1, 1  |            state_update #B2 (is_terminal=1) <-----| 1, 1  | // B replies with a new `state_update` which signs `1, 1` this time
        | 1, 1  |----------> state_update #A2 (is_terminal=1)       | 1, 1  | // A replies with a new `state_update` which signs `1, 1` this time
        | 1, 1  |<---------- state_update #B2 (is_terminal=1)       | 1, 1  | // A gets a new `state_update` which correctly signs `1, 1` and is terminal, B starts resolving A's HTLC
        | 1, 1  |            state_update #A2 (is_terminal=1) ----->| 1, 1  | // B gets a new `state_update` which correctly signs `1, 1` and is terminal, A starts resolving B's HTLC
        |       |                                                   |       |
        |   A   |                                                   |   B   |
        |       |                                                   |       |
        +-------+                                                   +-------+

        Where B is Host and A is Client
        
### The `state_update` Message

1. type: 75529 (`state_update`)
2. data:
  * [`u32`:`block_day`]
  * [`u32`:`local_updates`]
  * [`u32`:`remote_updates`]
  * [`u64`:`rate`]
  * [`signature`:`local_sig_of_remote`]
  * [`boolean`:`is_terminal`]
  
### Rationale

* Local `state_update` contains a _local_ signature of _remote_ view of next `last_cross_signed_state`. Signature is constructed as follows: generate a next local `last_cross_signed_state` with empty signature fields, then invert it (i.e. make a copy with `local_balance_msat` = `remote_balance_msat`, `local_updates` = `remote_updates` and so on), then sign its signature hash.

* In normal channels each peer keeps track of `local_next_htlc_id`/`remote_next_htlc_id` counters which are increased by incoming and outgoing `update_add_htlc` only, in hosted channels each peer keeps track of `local_updates`/`remote_updates` counters which are updated by every incoming and outgoing update message (`update_add_htlc`, `update_fulfill_htlc`, `update_fail_htlc`, `update_fail_malformed_htlc`).

* `rate` is Millisats/USD value. The field is used by client to reconstruct next crossigned state when the message recieved from the host. Host should ignore the field from the client.

* While verifying a signature a drift of 1 blockday is permitted i.e. `abs(local block_day - remote block_day) <= 1`.

* `is_terminal` is a flag which must always be set to `0` after sending local `update_add_htlc`/`update_fulfill_htlc`/`update_fail_htlc`/`update_fail_malformed_htlc` messages and always be set to `1` when replying to remote `state_update`. By using this convention peers can determine if remote peer is sending more updates or this is it for now and resolving should be started.

## Failure and state overriding

Normal channel operation may be interrupted in a number of ways (incorrect state update numbers, signature, timed out outgoing HTLC etc). Once this happens a peer must put a channel in `SUSPENDED` state and send out an `Error` message. Hosted channel use tagged errors with first two bytes of `Error` message reserved for the following cases:

`0001`: Wrong blockday in a remote message.  
`0002`: Wrong local signature from remote message.  
`0003`: Wrong remote signature from remote message.  
`0005`: Too many `state_update` messages without reaching of new local `last_cross_signed_state` (more than 16 in a row).  
`0006`: Timed out outgoing HTLC.  
`0007`: Remote peer has lost all upstream channels and can't resolve in-flight routed HTLCs.  
`0008`: Hosted channel denied by Host when Client was trying to invoke it.  

Normal operation may be resumed after channel gets `SUSPENDED` by Host sending a `state_override` message to Client which would erase all previous problematic state and set a new agreed upon balance breakdown. Client must manually accept this message which would send a `state_update` in return. Client's wallet UI/UX must be especially explicit about what is going on in this situation.

        +-------+                                           +-------+
        |       |<---------- state_override ----------------|       | // A has new local `last_cross_signed_state`
        |   A   |----------- state_update ----------------->|   B   | // B has new local `last_cross_signed_state`
        +-------+                                           +-------+

        Where B is Host and A is Client

### The `state_override` Message

1. type: 75527 (`state_override`)
2. data:
  * [`u32`:`block_day`]
  * [`u64`:`local_balance_msat`]
  * [`u32`:`local_updates`]
  * [`u32`:`remote_updates`]
  * [`u64`:`rate`]
  * [`signature`:`local_sig_of_remote`]

## Data format for channel state snapshot

When resolving disputes after a channel got `SUSPENDED` it may be necessary for a Client to manually provide a channel state snapshot which contains last cross signed state along with subsequent updates from both peers, those may include `update_fulfill_htlc` messages with preimages which can be used by Host to manually fulfill outstanding HTLCs. Otherwise  channel snapshot must not contain any other types of secrets.

### The `hosted_state` Message

1. data:
  * [`channel_id`:`channel_id`]
  * [`u16`:`num_next_local_updates`]
  * [`num_next_local_updates`*[`update_add_htlc`/`update_fulfill_htlc`/`update_fail_htlc`/`update_fail_malformed_htlc`]:`next_local_updates`]
  * [`u16`:`num_next_remote_updates`]
  * [`num_next_remote_updates`*[`update_add_htlc`/`update_fulfill_htlc`/`update_fail_htlc`/`update_fail_malformed_htlc`]:`next_remote_updates`]
  * [`last_cross_signed_state`:`last_cross_signed_state`]
  

## Resolving edge cases

### Peer misses a channel or falls behind

This situation is to be resolved in a standard way described in [`last_cross_signed_state` Rationale](https://github.com/btcontract/hosted-channels-rfc/blob/master/README.md#rationale-2).

### Receiver stops responding while incoming payment is in-flight

__Issue__: malicious or offline peer may accept `update_add_htlc` and a following `state_update` without ever replying. In this situation sender has a limited time to either fulfill a payment downstream (which would require obtaining a preimage from non-responding peer) or fail it, otherwise sender risks downstream channel getting remotely force-closed.

__Solution__: similar to how this is handled in normal channels, right before CLTV timelock is about to expire a sender must put a hosted channel into `SUSPENDED` mode and fail a payment upstream. Hosted channel can be put back to operational mode at a later time by exchaning `state_override` messages where receiver balance is adjusted such that failed in-flight HTLC is removed.

### Sender stops responding on obtaining a preimage

__Issue__: on receiving `update_fulfill_htlc` sender may use a preimage to fulfill a payment upstream and stop responding to receiver. Sender would thus get the funds into its upstream channel while receiver won't have an updated `last_cross_signed_state` with payment resolved.

__Solution__: the goal here is not to enforce payment receiving since this is impossible for hosted channels but to cryptographically prove that malicious behaviour has taken place on sender side. This is achieved in a following way:

1. When sender transmits `update_add_htlc` to receiver and expects `update_fulfill_htlc` in return this can only mean that sender already has a committed in-flight payment upstream which can be unconditionally resolved once sender knows a preimage.

2. After receiving `update_add_htlc` followed by `state_update` receiver has a new `last_cross_signed_state` where sender has signed an obligation to provide `X` funds to receiver if receiver reveals a preimage within next `Y` blocks (a CLTV expiry delta).

3. When having this data along with preimage revealed (`update_fulfill_htlc` sent) a sender software must notify an owner if payment is not getting resolved within a reasonable time frame, but well before CLTV deadline (otherwise sender could claim receiver just was not cooperating so sender had to fail a payment on CLTV expiry). Sender is then expected to take action which can range from contacing receiver support to revealing a `last_cross_signed_state` along with preimage publically, thus giving sender no chance of denying that active in-flight HTLC exists and respected preimage is revealed. Note that for this to work a final CLTV window set by receiver software should be large enough, at least 285 blocks is recommended.

### Host decides to refund a channel on-chain using Client's `refund_scriptpubkey`

__Issue__: After doing that Client may show up with last `last_cross_signed_state` claiming that funds are still in a hosted channel.

__Solution__: Host must wait at least 1 blockday until broadcasting an on-chain refunding transaction, in that case it will be included in a block whose `blockday` is higher than the one contained in Client's `last_cross_signed_state`. This will prove that refund has happened after any last known channel activity. After refunding this way hosted channel data may be safelly removed from Host database.
