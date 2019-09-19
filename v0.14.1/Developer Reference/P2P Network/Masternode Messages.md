---
title: "Masternode Messages"
excerpt: ""
---
# Overview
The following network messages enable the masternode features built in to Dash.

![Overview Of P2P Protocol Masternode Request And Reply Messages](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-p2p-masternode-messages.png)

For additional details, refer to the Developer Guide [Masternode Sync](developer-guide#masternode-sync) and [Masternode Payment](developer-guide#masternode-payment) sections.

# ssc

The `ssc` message is used to track the sync status of masternode objects. This message is sent in response to sync requests for the list of masternodes (`dseg` message), masternode payments (`mnget` message), governance objects (`govsync` message), and governance object votes (`govsync` message).

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nItemID | int | Required | Masternode Sync Item ID
| 4 | nCount | int | Required | Number of items to sync

Sync Item IDs

| ID | Description | Response To
|------|--------------|---------------
| 2 | MASTERNODE_SYNC_LIST | _Deprecated following activation of DIP3 in Dash Core 0.13.0_<br><br>`dseg` message
| 3 | MASTERNODE_SYNC_MNW | _Deprecated following activation of DIP3 in Dash Core 0.13.0_<br><br>`mnget` message
| 10 | MASTERNODE_SYNC_GOVOBJ | `govsync` message
| 11 | MASTERNODE_SYNC_GOVOBJ_VOTE | `govsync` message with non-zero hash

The following annotated hexdump shows a `ssc` message. (The
message header has been omitted.)

~~~
02000000 ................................... Item ID: MASTERNODE_SYNC_LIST (2)
bf110000 ................................... Count: 4543
~~~

# mnauth

*Added in protocol version 70214 of Dash Core*

The `mnauth` message is sent by a masternode immediately after sending a `verack` message to authenticate that the sender is a masternode. It is only sent when the sender is actually a masternode.

The `mnauth` message signs a challenge that was previously sent via a `version` message. The challenge is signed differently depending on if the connection is inbound or outbound.

This is primarily used as a DoS protection mechanism to allow persistent connections between masternodes to remain open even if inbound connection limits are reached.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 32 | proRegTxHash | uint256 | Version of the message
| 96 | sig | byte[] | BLS signature, signed with the operator key of the masternode

The following annotated hexdump shows a `mnauth` message. (The
message header has been omitted.)

~~~
63cd3bf06404d78f80163afeb4b13e18
7dc1c1d04997ef04f1a2ecb3166dd004 ........... ProRegTx Hash

12f2706bc75e9cb14a9ebf1d93d177d5
f266ad2eddc49ad463810cb976a3e4bb
abffc96819c5030fd5a7601af9c8ee50
0feb066b38a48af1a31b7242bd814bab
91e2a887f963904f33af851ddc9167d5
66d6d3bd6c07e99091edd8867d0dd56e ........... Masternode BLS Signature (96 bytes)
~~~