# Extension Bolt ZZZ: Dynamic Commitments

Authors:
  * Olaoluwa Osuntokun <roasbeef@lightning.engineering>
  * Eugene Siegel <eugene@lightning.engineering>

Created: 2022-06-24

# Table of Contents

- [Introduction](#introduction)

- [Specification](#specification)
  * [Proposal Messages](#proposal-messages)
    + [`dyn_begin_propose`](#-dyn_begin_propose-)
    + [`dyn_propose`](#-dyn_propose-)
    + [`dyn_propose_reply`](#-dyn_propose_reply-)
  * [Reestablish](#reestablish)
    + [`channel_reestablish`](#-channel_reestablish-)
  * [Musig2-Taproot](#musig2-taproot)
    + [`Signing Phase`](#signing-phase)

# Introduction

TODO

# Specification

## Proposal Messages

Three new messages are introduced that are common to all dynamic commitment flows.
They let each side propose what they want to change about the channel.

### `dyn_begin_propose`

This message is sent when a node wants to begin the dynamic commitment negotiation
process. This is a signaling message, similar to `shutdown` in the cooperative close
flow.

1. type: 111 (`dyn_begin_propose`)
2. data:
   * [`32*byte`:`channel_id`]
   * [`byte`: `begin_propose_flags`]

Only the least-significant-bit of `begin_propose_flags` is defined, the `reject` bit.

#### Requirements

The sending node:
  - MUST set `channel_id` to a valid channel they have with the  
    receipient.
  - MUST set undefined bits in `begin_propose_flags` to 0.
  - MUST set the `reject` bit in `begin_propose_flags` if they are
    rejecting the
    dynamic commitment negotiation request.
  - MUST NOT send `update_add_htlc` messages after sending this unless
    one of the following is true:
    - dynamic commitment negotiation has finished
    - a `dyn_begin_propose` with the `reject` bit has been received.
    - a reconnection has occurred.
  - MUST only send one `dyn_begin_propose` during a single negotiation.
  - MUST fail to forward additional incoming HTLCs from the peer.

The receiving node:
  - if `channel_id` does not match an existing channel it has with the
    peer:
    - MUST close the connection.
  - if the `reject` bit is set, but it hasn't sent a `dyn_begin_propose`:
    - MUST send an `error` and fail the channel.
  - if an `update_add_htlc` is received after this point and negotiation hasn't
    finished or terminated:
    - MUST send an `error` and fail the channel.

#### Rationale

todo flow diagram

This has similar semantics to the `shutdown` message where the channel comes to a state
where updates may only be removed. The `reject` bit is necessary to avoid having to
reconnect in order to have a useable channel state again.

### `dyn_propose`

todo flow diagram

This message is sent when neither side owes the other either a `revoke_and_ack` or
`commitment_signed` message and each side's commitment has no HTLCs. For now, only the
`dust_limit_satoshis` and `recipients_new_self_delay` parameter are defined in
negotiation. After negotiation completes, commitment signatures will use these parameters.

1. type: 113 (`dyn_propose`)
2. data:
   * [`32*byte`:`channel_id`]
   * [`dyn_propose_tlvs`:`tlvs`]

1. `tlv_stream`: `dyn_propose_tlvs`
2. types:
    1. type: 0 (`dust_limit_satoshis`)
    2. data:
        * [`u64`:`senders_new_dust_limit`]
    1. type: 2 (`to_self_delay`)
    2. data:
        * [`u16`:`recipients_new_self_delay`]

#### Requirements

The sending node:
  - MUST set `channel_id` to an existing one it has with the recipient.
  - MUST NOT set `dust_limit_satoshis` to a value that trims any currently untrimmed
    output on the commitment transaction.
    - (NOTE) When other fields are introduced, MUST NOT set values that conflict with
      the current commitment transaction such as setting `channel_reserve` too high.
  - MUST NOT send a `dyn_propose` if a prior one is waiting for `dyn_propose_reply`.
  - MUST remember its last sent `dyn_propose` parameters.
  - MUST send this message as soon as both side's commitment transaction is free of
    any HTLCs and both sides have sent `dyn_begin_propose`.

The receiving node:
  - if `channel_id` does not match an existing channel it has with the sender:
    - MUST close the connection.
  - if it does not agree with a parameter:
    - MUST send a `dyn_propose_reply` with the `reject` bit set.
      - (use TLV to tell _what_ param??)
  - else:
    - MUST send a `dyn_propose_reply` without the `reject` bit set.

#### Rationale

The requirement to not allow trimming outputs is just to make the dynamic commitment
flow as uninvasive as possible to the commitment transaction. A similar requirement
should be added for any new parameter such as the `channel_reserve`.

The requirement for a node to remember what it last _sent_ and for it to remember
what it _accepted_ is necessary to recover on reestablish. See the reestablish section
for more details.

### `dyn_propose_reply`

todo - tell what params are rejected in tlvs?

This message is sent in response to a `dyn_propose`. It may either accept or reject
the `dyn_propose`. If it rejects a `dyn_propose`, it allows the counterparty to send
another `dyn_propose` to try again. If for some reason, negotiation is taking too long,
it is possible to exit this phase by reconnecting as long as the exiting node hasn't
sent `dyn_propose_reply` without the `reject` bit.

1. type: 115 (`dyn_propose_reply`)
2. data:
   * [`32*byte`:`channel_id`]
   * [`byte`: `propose_reply_flags`]

The least-significant bit of `propose_reply_flags` is defined as the `reject` bit.

#### Requirements

The sending node:
  - MUST set `channel_id` to a valid channel they have with the recipient.
  - MUST set undefined bits in `propose_reply_flags` to 0.
  - MUST set the `reject` bit in `propose_reply_flags` if they are rejecting the newest
    `dyn_propose`.
  - MUST NOT send this message if there is no outstanding `dyn_propose` from the
    counterparty.
  - if the `reject` bit is not set:
    - MUST remember the related `dyn_propose` parameters and the local and remote commitment 
      heights for the next `propose_height`.

The receiving node:
  - if `channel_id` does not match an existing channel it has with the peer:
    - MUST close the connection.
  - if there isn't an outstanding `dyn_propose` it has sent:
    - MUST send an `error` and fail the channel.
  - if the `reject` bit was set:
    - MUST forget its last sent `dyn_propose` parameters.

A node:
  - once it has both sent and received `dyn_propose_reply` without the `reject` bit set:
    - MUST increment their `propose_height`.

#### Rationale

The `propose_height` starts at 0 for a channel and is incremented by 1 every time the
dynamic commitment proposal phase completes for a channel. See the reestablish section
for why this is needed.

## Reestablish

### `channel_reestablish`

A new TLV that denotes the node's current `propose_height` is included.

1. `tlv_stream`: `channel_reestablish_tlvs`
2. types:
    1. type: 20 (`propose_height`)
    2. data:
        * [`u64`:`propose_height`]

#### Requirements

The sending node:
  - MUST set `propose_height` to the number of dynamic proposal negotiations it has
    completed. The point at which it is incremented is described in the `dyn_propose_reply`
    section. 

The receiving node:
  - if the received `propose_height` equals its own `propose_height`:
    - MUST forget any stored proposal state for `propose_height`+1 in case negotiation didn't
      complete. Can continue using the channel.
    - SHOULD forget any state that is unnecessary for heights <= `propose_height`.
  - if the received `propose_height` is 1 greater than its own `propose_height`:
    - if it does not have any remote parameters stored for the received `propose_height`:
      - MUST send an `error` and fail the channel. The remote node is either lying about the
        `propose_height` or the recipient has lost data since its not possible to advance the
        height without the recipient storing the remote's parameters.
    - resume using the channel with its last-sent `dyn_propose` and the stored `dyn_propose`
      parameters and increment its `propose_height`.
  - if the received `propose_height` is 1 less than its own `propose_height`:
    - resume using the channel with the new paramters.
  - else:
    - MUST send an `error` and fail the channel. State was lost.

#### Rationale

If both sides have sent and received `dyn_propose_reply` without the `reject` bit before the
connection closed, it is simple to continue. If one side has sent and received
`dyn_propose_reply` without the `reject` bit and the other side has only sent `dyn_propose_reply`,
the flow is recoverable on reconnection as the side that hasn't received `dyn_propose_reply` knows
that the other side accepted their last sent `dyn_propose` based on the `propose_height` in the
reestablish message.

## Musig2 Taproot

This section describes how dynamic commitments can upgrade regular channels to simple
taproot channels. The regular dynamic proposal phase is executed followed by a signing phase.
A channel-type will be included in `dyn_propose` and both sides must agree on it. The
funder of the channel will also propose a fee to use for the kickoff transaction. The fundee
may reject it in `dyn_propose_reply`. In the regular dynamic proposal phase, the parameters
that modify the commitment aren't signed for (i.e. `to_self_delay`), however they will be
when exchanging commitment signatures to upgrade a channel to the simple-taproot-channel type.
The `dyn_propose_reply` message will contain two nonces for use in signing from the musig2
output.

NOTE: The design of the protocol here does not allow upgrading a channel to taproot and then,
without having the intermediate transactions confirming on-chain, upgrading to another future
commitment format.

Extensions to `dyn_propose` and `dyn_propose_reply`:

1. `tlv_stream`: `dyn_propose_tlvs`
2. types:
    1. type: 4 (`channel_type`)
    2. data:
        * [`...*byte`:`type`]
    1. type: 6 (`kickoff_fee`)
    2. data:
        * [`u32`:`kickoff_feerate_per_kw`]

1. `tlv_stream`: `dyn_propose_reply_tlvs`
2. types:
    1. type: 0 (`local_musig2_pubnonce`)
    2. data:
        * [`66*byte`:`nonces`]
    1. type: 2 (`remote_musig2_pubnonce`)
    2. data:
        * [`66*byte`:`nonces`]

The funder MUST only send `kickoff_fee` if they can pay for `kickoff_fee`+660 using their
own output while adhering to the `channel_reserve` restriction. The fundee MUST reject
via `dyn_propose_reply` if the funder cannot pay for the kickoff.

In the case that the connection is lost after the proposal phase has completed, but before
any commitment signatures are sent, the `channel_reestablish` message MUST contain two nonces.
If a node has sent `dyn_propose_reply` without the `reject` bit, it will persist the
`channel_type` per the requirements. If a node starts up and sees it has persisted a `channel_type`
of a simple-taproot-channel from the `dyn_propose_reply` parameters, it MUST send nonces in
`channel_reestablish`. Otherwise, neither side would be able to send commitment signatures.

### Signing Phase

To upgrade a regular channel to a simple-taproot-channel, the original funding output
must spend to a intermediate, kick-off transaction using a kickoff signature that pays to
a v1 witness script with a musig2 key as the internal key. Nonces are included in messages
per the simple-taproot proposal.

#### commit_sig

The commit_sig does not change, but adds a nonce in the TLV section per the simple-taproot
channel proposal. It changes what it signs in the following ways:

- `txin[0]` outpoint is the outpoint of the kickoff transaction's new musig2 funding outpoint.
- Because the new musig2 funding outpoint has a smaller amount, the funder's `to_local`
  balance is reduced. The `dyn_propose` stage won't complete if the funder cannot pay for the
  kickoff transaction's fees on either commitment transaction.

1. `tlv_stream`: `commit_sig_tlvs`
2. types:
    1. type: 2 (`remote_musig2_pubnonce`)
    2. data:
        * [`66*byte`:`nonces`]

#### kickoff_sig

The kickoff_sig is a signature that is combined with the corresponding one from the remote
party to spend from the original funding outpoint into the new musig2 output. To keep things
simple, no additional inputs are added to the intermediate transaction. An anchor output is
attached for either side. See
https://github.com/lightning/bolts/pull/863#pullrequestreview-1023605285 for more context.
An anchor output is required at the kickoff-tx level so that CPFP Carve Out still works. The
semantics of carve out are such that the "carve-out" transaction must only have one ancestor.
Therefore, using the commitment transaction's anchor as a carve-out wouldn't work since it
would have an ancestor of the commitment transaction and of the kickoff transaction. Using
anchors at the kickoff transaction level makes the commitment transaction anchors redundant
in adversarial scenarios, but they are still used to keep the commitment transaction format
consistent. Each side will ALWAYS have an anchor on the kickoff transaction.

1. type: 777 (`kickoff_sig`)
2. data:
   * [`32*byte`:`channel_id`]
   * [`signature`:`signature`]

The `kickoff_sig` message signs the below transaction.

Kickoff transaction format:

* version: 2
* locktime: 0
* txin count: 1
  * `txin[0]` outpoint: `txid` and `output_index` from the `funding_created` message
  * `txin[0]` sequence: 0
  * `txin[0]` script bytes: 0
  * `txin[0]` witness: `0 <signature_for_pubkey1> <signature_for_pubkey2>`

`new_funding_output`:
  * `txout[0]` amount: `txin[0].amount - 660 - kickoff_feerate_per_kw*kickoff_transaction_weight/1000`
  
This is a version 1 witness script:
  * `OP_1 funding_key`
  * where:
    * `funding_key = combined_funding_key + tagged_hash("TapTweak", combined_funding_key)*G`
    * `combined_funding_key = musig2.KeyAgg(musig2.KeySort(pubkey1, pubkey2))`
  
The funding keys are reused here, but this should not be a big deal.

`anchor_output_1`
  * `txout[1]` amount: 330

This is a version 1 witness script:
  * `OP_1 anchor_output_key`
  * where:
    * `anchor_internal_key = local_funding_pubkey/remote_funding_pubkey`
    * `anchor_output_key = anchor_internal_key + tagged_hash("TapTweak", anchor_internal_key || anchor_script_root)`
    * `anchor_script_root = tapscript_root([anchor_script])`
    * `anchor_script`:
        ```
        OP_16 OP_CHECKSEQUENCEVERIFY
        ```

`anchor_output_2`
  * Idential to `anchor_output_1` except the key is for the opposite party.

Weight:
  * p2tr: 34 bytes
    - OP_1: 1 byte
    - OP_DATA: 1 byte (witness_script_SHA256 length)
    - witness_script_SHA256: 32 bytes

  * witness_header: 2 bytes
    - flag: 1 byte
    - marker: 1 byte

  * funding_output_script: 71 bytes
    - OP_2: 1 byte
    - OP_DATA: 1 byte (pub_key_alice length)
    - pub_key_alice: 33 bytes
    - OP_DATA: 1 byte (pub_key_bob length)
    - pub_key_bob: 33 bytes
    - OP_2: 1 byte
    - OP_CHECKMULTISIG: 1 byte

  * funding_input_witness: 222 bytes
    - number_of_witness_elements: 1 byte
    - nil_length: 1 byte
    - sig_alice_length: 1 byte
    - sig_alice: 73 bytes
    - sig_bob_length: 1 byte
    - sig_bob: 73 bytes
    - witness_script_length: 1 byte
    - witness_script: 71 bytes (funding_output_script)

  * kickoff_txin_0: 41 bytes (excl. witness)
    - previous_out_point: 36 bytes
      - hash: 32 bytes
      - index: 4 bytes
    - var_int: 1 byte (script_sig length)
    - script_sig: 0 bytes
    - witness: SOLD SEPARATELY
    - sequence: 4 bytes

  * musig2_funding_output: 43 bytes
    - value: 8 bytes
    - var_int: 1 byte (pk_script length)
    - pk_script (p2tr): 34 bytes

  * anchor_output: 43 bytes
    - value: 8 bytes
    - var_int: 1 byte (pk_script length)
    - pk_script (p2tr): 34 bytes

  * kickoff_transaction: 180 bytes (excl. witness)
    - version: 4 bytes
    - witness_header: SOLD SEPARATELY
    - count_tx_in: 1 byte
    - tx_in: 41 bytes
      - kickoff_txin_0: 41 bytes
    - count_tx_out: 1 byte
    - txout: 129 bytes
      - musig2_funding_output: 43 bytes
      - anchor_output_local: 43 bytes
      - anchor_output_remote: 43 bytes
    - lock_time: 4 bytes

  - Multiplying non-witness data by 4 gives a weight of:
    - kickoff_transaction_weight = kickoff_transaction * 4 = 720WU
  - Adding the witness data:
    - kickoff_transaction_weight += (funding_input_witness + witness_header)
    - kickoff_transaction_weight = 944WU

#### revoke_and_ack

Per the simple-taproot-channel proposal, the `revoke_and_ack` message contains a single pair
of nonces used by the recipient to construct a commitment transaction for the sender.

1. `tlv_stream`: `revoke_and_ack_tlvs`
2. types:
    1. type: 2 (`local_musig2_pubnonce`)
    2. data:
        * [`66*byte`:`nonces`]

#### The message flow to upgrade a channel to simple-taproot:

        +-------+                               +-------+
        |       |--(1)------ commit_sig ------->|       |
        |       |                               |       |
        |       |<-(2)------ commit_sig --------|       |
        |   A   |--(3)----- kickoff_sig ------->|   B   |
        |       |                               |       |
        |       |<-(4)----- kickoff_sig --------|       |
        |       |<-(5)----- rev_and_ack --------|       |
        |       |--(6)----- rev_and_ack ------->|       |
        +-------+                               +-------+

The ordering here is important. If the sending of kickoff signatures was first, then one
side could choose to not send their kickoff signature and wait to receive the counterparty's. Then
once it's received, broadcast the intermediate kick-off transaction that neither side
can claim since commitment signatures weren't exchanged first. The kickoff signatures MUST only
be sent if a `commit_sig` has been sent and received.

The `revoke_and_ack` messages are intentionally last. They must only be sent if an `kickoff_sig`
has been received. Otherwise the following flow means that A cannot force close its own channel:

        +-------+                               +-------+
        |       |--(1)------ commit_sig ------->|       |
        |       |                               |       |
        |       |<-(2)------ commit_sig --------|       |
        |   A   |--(3)----- kickoff_sig ------->|   B   |
        |       |                               |       |
        |       |--(4)----- rev_and_ack ------->|       |
        +-------+                               +-------+

Including `revoke_and_ack` in the signing phase means that it is possible to determine when the flow
is done upon a subsequent reconnect by examining the `next_revocation_number`s sent and received in
`channel_reestablish`.

#### Continue or fail

Once the proposal phase is complete, it is not possible to exit without completing the
signing stage. This is by design so that one side cannot lie about not receiving a commitment
signature, such as in the below scenario:

        +-------+                               +-------+
        |       |--(1)----- dyn_propose ------->|       |
        |       |                               |       |
        |       |<-(2)----- dyn_propose --------|       |
        |   M   |--(3)--- dyn_propose_reply --->|   B   |
        |       |                               |       |
        |       |?-(4)--- dyn_propose_reply ----|       |
        |       |?-(5)----- commit_sig ---------|       |
        +-------+                               +-------+

Note that on restart B is not able to tell if M received `dyn_propose_reply` and `commit_sig`.
If a total restart was allowed, then M might be able to propose N different `to_self_delay`
and receive signatures for N different commitment transactions. Then, M could complete the signing
phase up until receiving an `kickoff_sig` and broadcast any one of the N commitment transactions.
To avoid an implementation keeping track of N valid commitments, we disallow restarting
the flow.

This is avoided by _requiring_ that if one side has sent and received `dyn_propose_reply`,
the negotiation MUST finish or the channel is force closed. Before this point, neither
node can determine whether negotiation is done.

### Reestablish Taproot

On reestablish things are simple. The signature MUST sign the exact same commitment
transaction, but will use different nonces. The kickoff signature MUST also sign the same
transaction each time. Nonces will be sent if the saved, proposed `channel_type` indicates
that taproot was negotiated in the `dyn_propose` step.

No matter where the nodes were in the signing phase, it starts at the beginning on reconnect.
This simplifies the flow. If a duplicate `commitment_signed`, `kickoff_sig`, or `revoke_and_ack`
is received, everything except the nonces can be ignored. This is unlike the regular channel
update flow, but since each party knows it is performing dynamic commitment negotiation,
duplicate messages can be tolerated.

The signing phase is complete when a node receives a `channel_reestablish` with a
`next_revocation_number` that is greater than their local commitment number at the start of
the dynamic proposal flow AND when its sent `next_revocation_number` is greater than the remote
commitment number at the start of the dynamic proposal flow. This requires each side to
remember the starting commitment heights when acknowleding and persisting received
`dyn_propose` parameters.
