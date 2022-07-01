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
A channel-type will be included in `dyn_propose` and both sides must agree on it. In the
regular dynamic proposal phase, the parameters that modify the commitment aren't signed for
(i.e. `to_self_delay`), however they will be when exchanging commitment signatures to upgrade a
channel to the simple-taproot-channel type.

Extensions to `dyn_propose`:

1. `tlv_stream`: `dyn_propose_tlvs`
2. types:
    1. type: 4 (`channel_type`)
    2. data:
        * [`...*byte`:`type`]

Negotiation should begin on reestablish. Each node will know that they are negotiating a
simple-taproot-channel if the stored `channel_type` is of the simple-taproot-channel type.

### Signing Phase

The only change is that the outputs of the commitment transaction spend to v1 witness scripts
rather than v0 witness scripts. Unlike un-upgraded simple-taproot-channels, dynamically
upgraded channels do not use musig2 and therefore do not need to exchange nonces.

#### commit_sig

The commit_sig does not change, but will sign for upgraded witness scripts for all outputs.
The fee may change depending on the type of the original channel. If upgrading from an anchors
channel, the fee is identical since it is a simple v0->v1 mapping. If upgrading from a pre-
anchors channel, then anchor outputs will be added and the `to_remote` output will go from a
p2wpkh->p2tr output which has a slightly higher weight.

The fundee should reject the dynamic commitment proposal if the funder cannot pay for the
upgrade. The funder should likewise reject a dynamic commitment proposal from the fundee if
they cannot afford to upgrade to a simple-taproot-channel. This should happen during the
`dyn_propose/dyn_propose_reply` phase.

#### revoke_and_ack

This message is used during the flow to ensure that once negotiation is complete, the channel
has been irrevocably upgraded to a non-musig2 simple-taproot-channel.

#### The message flow to upgrade a channel to simple-taproot:

        +-------+                               +-------+
        |       |--(1)------ commit_sig ------->|       |
        |       |                               |       |
        |       |<-(2)------ commit_sig --------|       |
        |   A   |                               |   B   |
        |       |<-(5)----- rev_and_ack --------|       |
        |       |--(6)----- rev_and_ack ------->|       |
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
and receive signatures for N different commitment transactions. To avoid an implementation
keeping track of N valid commitments, we disallow restarting the flow.

This is avoided by _requiring_ that if one side has sent and received `dyn_propose_reply`,
the negotiation MUST finish or the channel is force closed. Before this point, neither
node can determine whether negotiation is done.

### Reestablish Taproot

On reestablish things are simple. The signature MUST sign the exact same commitment
transaction.

No matter where the nodes were in the signing phase, it starts at the beginning on reconnect.
This simplifies the flow. If a duplicate `commitment_signed` or `revoke_and_ack` is received,
it can be ignored. This is unlike the regular channel update flow, but since each party knows
it is performing dynamic commitment negotiation, duplicate messages can be tolerated.

The signing phase is complete when a node receives a `channel_reestablish` with a
`next_revocation_number` that is greater than their local commitment number at the start of
the dynamic proposal flow AND when its sent `next_revocation_number` is greater than the remote
commitment number at the start of the dynamic proposal flow. This requires each side to
remember the starting commitment heights when acknowleding and persisting received
`dyn_propose` parameters.
