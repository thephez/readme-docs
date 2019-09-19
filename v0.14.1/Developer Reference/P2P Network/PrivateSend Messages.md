---
title: "PrivateSend Messages"
excerpt: ""
---
# Overview

The following network messages all help control the PrivateSend (formerly DarkSend) coin mixing features built in to Dash and facilitated by the masternode network.

Since the messages are all related to a single process, this diagram shows them sequentially numbered. The `dssu` message (not shown) is sent by the masternode in conjunction with some responses. For additional details, refer to the Developer Guide [PrivateSend section](developer-guide#privatesend).

![Overview Of P2P Protocol PrivateSend Request And Reply Messages](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-p2p-privatesend-messages.png)

# dsa

The `dsa` message allows a node to join a mixing pool. A collateral fee is required and may be forfeited if the client acts maliciously.  The message operates in two ways:

1. When sent to a masternode without a current mixing queue, it initiates the
  start of a new mixing queue

2. When sent to a masternode with a current mixing queue, it attempts to join the
  existing queue

Dash Core attempts to join an existing queue first and only requests a new one if no existing ones are available.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nDenom | int | Required | Denomination that will be exclusively used when submitting inputs into the pool
| 216+ | txCollateral | `tx` message | Required | Collateral TX that will be charged if this client acts maliciously

The following annotated hexdump shows a `dsa` message. (The message header has been omitted.) Note that the 'Required inputs' bytes will only be preset if Spork 6 is active and protocol version => 70209.

~~~
02000000 ................................... Denomination: 1 Dash (2)

Collateral Transaction
| Previous Output
| |
| | 010000000183bd1980c71c38f035db9b
| | 14d7f934f7d595181b3436e362899026 ....... Outpoint TXID
| | 19f3f7d3 ............................... Outpoint index number: 3556242201
|
| 83 ....................................... Bytes in sig. script: 131
|
| 000000006b483045022100f4d8fa0ae4132235fe
| cd540a62715ccfb1c9a97f8698d066656e30bb1e
| 1e06b90220301b4cc93f38950a69396ed89dfcc0
| 8e72ec8e6e7169463592a0bf504946d98b812102
| fa4b9c0f9e76e06d57c75cab9c8368a62a1ce8db
| 6eb0c25c3e0719ddd9ab549cffffffff01e09304
| 00000000001976a914f895 ................... Secp256k1 signature: None
|
| 6a4eb0e5 ................................. Sequence number: 3853536874
~~~


# dsc

The `dsc` message indicates a PrivateSend mixing session is complete.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nSessionID | int | Required | ID of the mixing session
| 4 | nMessageID | int | Required | ID of the message describing the result of the mixing session

Reference the Message IDs table under the `dssu` message for descriptions of the Message ID values.

The following annotated hexdump shows a `dsc` message. (The message header has been omitted.)

~~~
d9070700 ............................. Session ID: 791686
14000000 ............................. Message ID: MSG_SUCCESS (20)
~~~


# dsf

The `dsf` message is sent by the masternode as the final mixing transaction in a PrivateSend mixing session. The masternode expects nodes in the mixing session to respond with a `dss` message.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nSessionID | int | Required | ID of the mixing session
| # | txFinal | `tx` message | Required |  Final mixing transaction with unsigned inputs

The following annotated hexdump shows a `dsf` message. (The message header has been omitted.) Transaction inputs/outputs are only shown for a single node (compare with the `dsi` message and `dss` message hexdumps).

~~~
86140c00 ............................. Session ID: 791686

Transaction Message
| 01000000 ................................. Version: 1
|
| 0f ......................................... Number of inputs: 15
|
| [...] ...................................... 5 transaction inputs omitted
|
| Transaction input #6
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 02000000 ................................. Outpoint index number: 0
| |
| | 00 ....................................... Bytes in sig. script: 0
| | .......................................... Secp256k1 signature: None
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| Transaction input #7
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 0f000000 ................................. Outpoint index number: 15
| |
| | 00 ....................................... Bytes in sig. script: 0
| | .......................................... Secp256k1 signature: None
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| Transaction input #8
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 0d000000 ................................. Outpoint index number: 13
| |
| | 00 ....................................... Bytes in sig. script: 0
| | .......................................... Secp256k1 signature: None
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
|
| [...] ...................................... 7 more transaction inputs omitted
|
|
| 0f ......................................... Number of outputs: 15
|
| Transaction output #1
| | e8e4f50500000000 ......................... Duffs (1.00001 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | 14826d7ba05cf76588a5503c03951dc9
| | | | 14c91b6c ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
|
| [...] ...................................... 3 transaction outputs omitted
|
|
| Transaction output #5
| | e8e4f50500000000 ......................... 100,001,000 Duffs (1.0001 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | 426614716e94812d483bca32374f6ac8
| | | | cd121b0d ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
|
| [...] ...................................... 9 transaction outputs omitted
|
|
| Transaction output #15
| | e8e4f50500000000 ......................... 100,001,000 Duffs (1.0001 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | f01197177de2358928196a543b2bbd97
| | | | 3c2ab002 ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
| 00000000 ................................... locktime: 0 (a block height)
~~~


# dsi

The `dsi` message replies to a `dsq` message that has the Ready field set to 0x01. The `dsi` message contains user inputs for mixing along with the outputs and a collateral. Once the masternode receives `dsi` messages from all members of the pool, it responds with a `dsf` message.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| ? | vecTxDSIn | CTxDSIn[] | Required | Vector of users inputs (CTxDSIn serialization is equal to CTxIn serialization)
| 216+ | txCollateral | `tx` message | Required | Collateral transaction which is used to prevent misbehavior and also to charge fees randomly
| ? | vecTxDSOut | CTxDSOut[] | Required | Vector of user outputs (CTxDSOut serialization is equal to CTxOut serialization)

The following annotated hexdump shows a `dsi` message. (The message header has been omitted.)

~~~
User inputs
| 03 ......................................... Number of inputs: 3
|
| Transaction input #1
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 02000000 ................................. Outpoint index number: 2
| |
| | 00 ....................................... Bytes in sig. script: 0
| | .......................................... Secp256k1 signature: None
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| Transaction input #2
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 0f000000 ................................. Outpoint index number: 15
| |
| | 00 ....................................... Bytes in sig. script: 0
| | .......................................... Secp256k1 signature: None
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| Transaction input #3
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 0d000000 ................................. Outpoint index number: 13
| |
| | 00 ....................................... Bytes in sig. script: 0
| | .......................................... Secp256k1 signature: None
| |
| | ffffffff ................................. Sequence number: UINT32_MAX

Collateral Transaction
| 01000000 ................................... Version: 1
|
| 01 ......................................... Number of inputs: 1
|
| Previous Output
| |
| | 83bd1980c71c38f035db9b14d7f934f7
| | d595181b3436e36289902619f3f7d383 ......... Outpoint TXID
| | 00000000 ................................. Outpoint index number: 0
| |
| | 6b ....................................... Bytes in sig. script: 107
| |
| | 483045022100f4d8fa0ae4132235fecd540a
| | 62715ccfb1c9a97f8698d066656e30bb1e1e
| | 06b90220301b4cc93f38950a69396ed89dfc
| | c08e72ec8e6e7169463592a0bf504946d98b
| | 812102fa4b9c0f9e76e06d57c75cab9c8368
| | a62a1ce8db6eb0c25c3e0719ddd9ab549c ....... Secp256k1 signature
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| 01 ......................................... Number of outputs: 1
|
| | e093040000000000 ......................... 300,000 Duffs (0.003 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | f8956a4eb0e53b05ee6b30edfd2770b5
| | | | 26c1f1bb ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
| 00000000 ................................... locktime: 0 (a block height)

User outputs
| 03 ......................................... Number of outputs: 3
|
| Transaction output #1
| | e8e4f50500000000 ......................... 100,001,000 Duffs (1.0001 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | 14826d7ba05cf76588a5503c03951dc9
| | | | 14c91b6c ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
| Transaction output #2
| | e8e4f50500000000 ......................... 100,001,000 Duffs (1.0001 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | f01197177de2358928196a543b2bbd97
| | | | 3c2ab002 ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
| Transaction output #3
| | e8e4f50500000000 ......................... 100,001,000 Duffs (1.0001 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | 426614716e94812d483bca32374f6ac8
| | | | cd121b0d ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
~~~


# dsq

The `dsq` message provides nodes with mixing queue details and notifies them when to sign final mixing TX messages.

If the message indicates the queue is not ready, the node verifies the message is valid. It also verifies that the masternode is not flooding the network with `dsq` messages in an attempt to dominate the queuing process. It then relays the message to its connected peers.

If the message indicates the queue is ready, the node responds with a `dsi` message.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nDenom | int | Required | Denomination allowed in this mixing session
| 36 | masternodeOutPoint | outPoint | Required | The unspent outpoint of the masternode (holding 1000 DASH) which is hosting this session
| 8 | nTime | int64_t | Required | Time this `dsq` message was created
| 1 | fReady | bool | Required | Indicates if the mixing pool is ready to be executed
| 97 | vchSig | char[] | Required | _ECDSA signature (65 bytes) prior to DIP3 activation_<br><br>BLS Signature of this message by masternode verifiable via pubKeyMasternode (Length (1 byte) + Signature (96 bytes))

Denominations (per [`src/privatesend.cpp`][privatesend denominations])

| Value | Denomination
|------|--------------
| 1 | 10 Dash
| 2 | 1 Dash
| 4 | 0.1 Dash
| 8 | 0.01 Dash
| 16 | 0.001 Dash

The following annotated hexdump shows a `dsq` message. (The message header has been omitted.) Note that the 'Required inputs' bytes will only be preset if Spork 6 is active and protocol version => 70209.

~~~
01000000 ............................. Denomination: 10 Dash (1)

Masternode Outpoint
| a383a2489aedccfab4bb41368d1c8ee3
| 10d9ee90cb3d181880ce4e0cdb36ecb7
| 0f000000 ........................... Outpoint index number: 15

10b4235c00000000 ..................... Create Time: 2018-12-26 17:02:08 UTC

00 ................................... Ready: 0

60 ................................... Signature length: 96

0409a1349869a02e90e6e1f6d92bf995
27a72542fed987f6d2719596973d89e6
74605a3585b1335650f1555f7576061d
110fb72b3308e378ac8e8fbebeeffdb4
9b2a6562ad965bb3c3fd3f8e68483fdb
0d1401e2264071a74fc01d51e943ce9f ..... Masternode BLS Signature
~~~


# dss

The `dss` message replies to a `dsf` message sent by the masternode managing the mixing session.  The `dsf` message provides the unsigned transaction inputs for all members of the mixing pool. Each node verifies that the final transaction matches what is expected. They then sign any transaction inputs belonging to them and then relay them to the masternode via this `dss` message.

Once the masternode receives and validates all `dss` messages, it issues a `dsc` message. If a node does not respond to a `dsf` message with signed transaction inputs, it may forfeit the collateral it provided. This is to minimize malicious behavior.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| # | inputs | txIn[] | Required | Signed inputs for mixing session

The following annotated hexdump shows a `dss` message. (The message header has been omitted.) Note that these will be the same transaction inputs that were supplied (unsiged) in the `dsi` message.

~~~
User inputs
| 03 ......................................... Number of inputs: 3
|
| Transaction input #1
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 02000000 ................................. Outpoint index number: 2
| |
| | 6b ....................................... Bytes in sig. script: 107
| | 483045022100b3a861dca83463aabf5e4a14a286
| | 1b9c2e51e0dedd8a13552e118bf74eb4a68d0220
| | 4a91c416768d27e6bdcfa45d28129841dbcc728b
| | f0bbec9701cfc4e743d23adf812102cc4876c9da
| | 84417dec37924e0479205ce02529bb0ba88631d3
| | ccc9cfcdf00173 ........................... Secp256k1 signature
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| Transaction input #2
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 0f000000 ................................. Outpoint index number: 15
| |
| | 6a ....................................... Bytes in sig. script: 106
| | 4730440220268f3b7799ca4ec132e511a4756019
| | c56016f7771561dc0597d84e9b1fa9fc08022067
| | 5199b9b3f9a7eba69b7bbb4aa2a413d955762f9d
| | 68be5a9c02c6772c8078fd812103258925f0dbbf
| | 9d5aa20a675459fa2e86c9f9061dee82a00dca73
| | 9080f051d891 ............................. Secp256k1 signature
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| Transaction input #3
| |
| | 36bdc3796c5630225f2c86c946e2221a
| | 9958378f5d08da380895c2656730b5c0 ......... Outpoint TXID
| | 0d000000 ................................. Outpoint index number: 13
| |
| | 6a ....................................... Bytes in sig. script: 106
| | 4730440220404bb067e0c94a2bd75c6798c1af8c
| | 95e8b92f5e437cff2bcb4660f24a34d06d02203a
| | b707bd371a84a9e7bd1fbe3b0c939fd23e0a9165
| | de78809b9310372a4b3879812103a9a6c5204811
| | a8cab04b595ed622a1fed6efd3b2d888fadd0c97
| | 3737fcdf2bc7 ............................. Secp256k1 signature
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
~~~


# dssu

The `dssu` message provides a mixing pool status update.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nMsgSessionID | int | Required | Session ID
| 4 | nMsgState | int | Required | Current state of mixing process
| 4 | nMsgEntriesCount | int | Required | _Deprecated in Dash Core 0.14.1_<br><br>Number of entries in the mixing pool
| 4 | nMsgStatusUpdate | int | Required | Update state and/or signal if entry was accepted or not
| 4 | nMsgMessageID | int | Required | ID of the typical masternode reply message

Pool State

| State | Description
|------|--------------
| 0 | `POOL_STATE_IDLE`
| 1 | `POOL_STATE_QUEUE`
| 2 | `POOL_STATE_ACCEPTING_ENTRIES`
| 3 | `POOL_STATE_SIGNING`
| 4 | `POOL_STATE_ERROR`
| 5 | `POOL_STATE_SUCCESS`

Pool Status Update

| Status | Description
|------|--------------
| 0 | `STATUS_REJECTED`
| 1 | `STATUS_ACCEPTED`

Message IDs

| Code | Description
|------|--------------
| 0x00 | `ERR_ALREADY_HAVE`
| 0x01 | `ERR_DENOM`
| 0x02 | `ERR_ENTRIES_FULL`
| 0x03 | `ERR_EXISTING_TX`
| 0x04 | `ERR_FEES`
| 0x05 | `ERR_INVALID_COLLATERAL`
| 0x06 | `ERR_INVALID_INPUT`
| 0x07 | `ERR_INVALID_SCRIPT`
| 0x08 | `ERR_INVALID_TX`
| 0x09 | `ERR_MAXIMUM`
| 0x0A (10) | `ERR_MN_LIST`
| 0x0B (11) | `ERR_MODE`
| 0x0C (12) | `ERR_NON_STANDARD_PUBKEY`
| 0x0D (13) | `ERR_NOT_A_MN` (Not used)
| 0x0E (14) | `ERR_QUEUE_FULL`
| 0x0F (15) | `ERR_RECENT`
| 0x10 (16) | `ERR_SESSION`
| 0x11 (17) | `ERR_MISSING_TX`
| 0x12 (18) | `ERR_VERSION`
| 0x13 (19) | `MSG_NOERR`
| 0x14 (20) | `MSG_SUCCESS`
| 0x15 (21) | `MSG_ENTRIES_ADDED`

The following annotated hexdump shows a `dssu` message. (The message header has been omitted.)

~~~
86140c00 ............................. Session ID: 791686
02000000 ............................. State: POOL_STATE_ACCEPTING_ENTRIES (2)
01000000 ............................. Status Update: STATUS_ACCEPTED (1)
13000000 ............................. Message ID: MSG_NOERR (0x13)
~~~


# dstx

The `dstx` message allows masternodes to broadcast subsidized transactions without fees (to provide security in mixing).

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| # | tx | `tx` message | Required | The transaction
| 36 | masternodeOutPoint | outPoint | Required | The unspent outpoint of the masternode (holding 1000 DASH) which is signing the message
| 97 | vchSig | char[] | Required | _ECDSA signature (65 bytes) prior to DIP3 activation_<br><br>BLS Signature of this message by masternode verifiable via pubKeyMasternode (Length (1 byte) + Signature (96 bytes))
| 8 | sigTime | int64_t | Require | Time this message was signed


The following annotated hexdump shows a `dstx` message. (The message header has been omitted.)

~~~
Transaction Message
| 0200 ....................................... Version: 2
| 0000 ....................................... Type: 0 (Classical Tx)
|
| 05 ......................................... Number of inputs: 5
|
| Transaction input #1
| |
| | 0adb782b2170018eada54534be880e70
| | 74ed8307a566731119b1782362af43ad ......... Outpoint TXID
| | 05000000 ................................. Outpoint index number: 5
| |
| | 6b ....................................... Bytes in sig. script: 107
| | 483045022100b1243fcba562a0f1d7c4
| | cc3b320645dfa96c6412f368ccdbc1b7
| | acb6b0aa1db502201606c81b0d79f52f
| | 47bcb071b64c37f72dd1378efa67a2de
| | dd86c44d393668fa812102d6ff581270
| | 632f5e972b0418ee871867b5c04b6eae
| | 3458ad135ad8f1daaa4fc2 ................... Secp256k1 signature
| |
| | ffffffff ................................. Sequence number: UINT32_MAX
|
| [...] ...................................... 4 more transaction inputs omitted
|
|
| 05 ......................................... Number of outputs: 5
|
| Transaction output #1
| | 10f19a3b00000000 ......................... Duffs (10.0001000 Dash)
| |
| | 19 ....................................... Bytes in pubkey script: 25
| | | 76 ..................................... OP_DUP
| | | a9 ..................................... OP_HASH160
| | | 14 ..................................... Push 20 bytes as data
| | | | 3eb7ae776b096231de9eca42dd57a677
| | | | d3b05452 ............................. PubKey hash
| | | 88 ..................................... OP_EQUALVERIFY
| | | ac ..................................... OP_CHECKSIG
|
| [...] ...................................... 4 more transaction outputs omitted
|
|
| 00000000 ................................... locktime: 0 (a block height)

Masternode Unspent Outpoint
| ccfbe4e7c220264cb0a8bfa5e91c6957
| 33c255384790e80e891a0d8f56a59d9e ......... Outpoint TXID
| 01000000 ................................. Outpoint index number: 1

60 ......................................... Signature length: 96

94c8e427f448789f58cda17445e76c64
d0efa7c089addcb378f9b8d04b72f499
a4e8e616b5011886b9cffcce29e17fc1
10ad8609c3ee1a3207b882e7ff58400f
42d6e6544108b349da2cc5e716a5f266
4a2dc96b0f080effd5349f2ae06ac234 .......... Masternode Signature

59b4235c00000000 .......................... Sig Time: 2018-12-26 17:03:21 UTC
~~~