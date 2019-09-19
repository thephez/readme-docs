---
title: "Quorum Messages"
excerpt: ""
---
The following network messages enable the long-living masternode quorum (LLMQ) features built in to Dash.

# Distributed Key Generation

The following network messages enable the creation of long living masternode quorums (LLMQs) as described in [DIP6](https://github.com/dashpay/dips/blob/master/dip-0006.md).

With the exception of the `qfcommit` message, these messages are for intra-quorum communication only and are not propagated on the Dash network.

## qcontrib

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qcontrib` message is used by each member of the DKG process to send key contributions to all other members.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | llmqType | uint8_t | The type of LLMQ
| 32 | quorumHash | uint256 | 	The quorum identifier
| 32 | proTxHash | uint256 | The ProRegTx hash of the complaining member
| 1-9 | vvecSize | compactSize uint | The size of the verification vector
| 48 * `vvecSize` | vvec | BLSPubKey[] | The verification vector
| 48 | ephemeralPubKey | BLSPubKey | Ephemeral BLS public key used to encrypt secret key contributions
| 32 | iv | uint256 | Initialization vector
| 1-9 | skCount | compactSize uint | Number of encrypted secret key contributions
| (1 + 32) * (`skCount`) | skContributions | byte[] | Secret key contributions encrypted to recipient masternodes’ BLS public operator key.<br><br>Each contribution consists of:<br>- Size: 1 byte<br>- Secret Key: 32 bytes
| 96 | sig | byte[] | BLS signature, signed with the operator key of the contributing masternode

More information can be found in the [Contribution phase section of DIP6](https://github.com/dashpay/dips/blob/master/dip-0006.md#2-contribution-phase).

The following annotated hexdump shows a `qcontrib` message. (The message header has been omitted.)

~~~
01  ........................................ LLMQ Type: 1 (LLMQ_50_60)

cb9a1552340175a8232437eb8ceceaea
4b90a0f75caff20ee12d230b00000000 ...........  Quorum Hash

cd1c97c52ccf163ee5dc264d411efc90
b07729cd34d9d2e7c7b3ca4b2a4e77cf ........... ProRegTx Hash

1e ......................................... Verification Vector Size: 30

Verification Vector (Truncated)
| 8da71ba5030e28c6c4de5e0eb1660d0f
| a9fd21ef4fef700a556f10286c9c34fb
| beb36fffb5b2a552a40d6c8e27aac338
| [...]
| 99d8649d226261162bcb5a11617d1732
| 553b8358d85b1d9e12a88eb3e979fb7c
| e49b5a21a82a74e9d06233199cb73db4 ......... Verification Vector (1440 bytes)

8d664929b596cdc8eb835d652944d61b
7fd21fd60ba0288af4f9e3a10658c8a8
56467082c728e2037791166705ada03a ........... Ephemeral BLS Public Key

93037a05b65adad6f5d44edc43500bff
71605f0e5f90ab92e3e0b46461c1c64d ........... IV Seed

32 ......................................... Contribution count: 50
Contributions
| Secret Key Contribution #1
| | 20 ..................................... Contribution Size: 32 bytes
| | | 31f3e8e5b2cc2063ee7fd1dd469dca12
| | | 4bdf506ee46fe825d5537aa3ce838225 ..... Encrypted Secret Key contribution
|
| Secret Key Contribution #2
| | 20 ..................................... Contribution Size: 32 bytes
| | | a6b3ff696ffc5e0c0a9b444c515edc48
| | | 5a9ccea0268c2a445fac5e24feda51a9 ..... Encrypted Secret Key contribution
|
| [...] .................................... 47 contributions omitted
|
| Secret Key Contribution #50
| | 20 ..................................... Contribution Size: 32 bytes
| | | 25f54cff411a577db9a416a60067f512
| | | 0750c77720207eb1484c90767b72faf8 ..... Encrypted Secret Key contribution

81f1003546f6735849c5691af93d324d
3a719fc4bb6d719907de3bce9833228e
648d03cd80666d70600fa8c936d30046
07bd444af3e494fb2a21273fcfa51986
3c4e139c67d2ffe0df07ac27ae63a0c8
e000da1aeda5f98ec9e64b801681bfc1 ........... BLS signature (Operator Key)
~~~


## qcomplaint

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qcomplaint` message is used to notify other members in the DKG process of any members that provided no contribution or an invalid secret key contribution. The notifications are divided into 2 fields:

 - `badMembers` - Sets a bit for each member that failed to provide a contribution
 - `complaints` - Sets a bit for each member that provided an invalid contribution

If a threshold number of quorum participants indicate a masternode didn't contribute, that masternode will be excluded from the quorum. Members that simply have a complaint against them are given an opportunity to respond (via a `qjustify` message) to attempt to prove to all participants that they did participate.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | llmqType | uint8_t | The type of LLMQ
| 32 | quorumHash | uint256 | 	The quorum identifier
| 32 | proTxHash | uint256 | The ProRegTx hash of the complaining member
| 1-9 | badBitSize | compactSize uint | Number of bits in the bad members bitvector
| (`badBitSize` + 7) / 8 | badMembers | byte[] | The bad members bitvector
| 1-9 | complaintsBitSize | compactSize uint | Number of bits in the complaints bitvector
| (`complaints`<br>`BitSize` + 7) / 8 | complaints | byte[] | The complaints bitvector
| 96 | sig | byte[] | BLS signature, signed with the operator key of the contributing masternode

More information can be found in the [Complaining phase section of DIP6](https://github.com/dashpay/dips/blob/master/dip-0006.md#3-complaining-phase).

The following annotated hexdump shows a `qcomplaint` message. (The message header has been omitted.)

~~~
01 ......................................... LLMQ Type: 1 (LLMQ_50_60)

b34b2bcb3430f403663e37be9c63c88e
4ca1f12c41846064cf960a0800000000 ........... Quorum Hash

b375607540bd9c6e4a5452d8c7a6a626
ec715222a0650321487843c79cac67d5 ........... ProRegTx hash

32 ......................................... Bad member bitvector size: 50
08800200004000 ............................. Bad members

32 ......................................... Complaints bitvector size: 50
00020080040000 ............................. Complaints

0639b0e8ccb667c161207ddc03183d4e
bb632eeb60f29e351963032a673abd61
3fb3e847dff78699481193cf385f0e08
0fdf518e26ef1e258b724408b1ee9d70
511696092b6c2ebfad5e24154a7f859f
0efe3fcb8d7042da624f7298876cc98e ........... BLS signature (Operator Key)
~~~


## qjustify

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qjustify` message is used to respond to complaints. This provides a way for nodes that have been complained about to offer proof of correct behavior. If a valid justification is not provided, all other nodes mark it as a bad. If a valid justification is provided, the complaining node is marked as bad instead (since it submitted a bad complaint).

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | llmqType | uint8_t | The type of LLMQ
| 32 | quorumHash | uint256 | 	The quorum identifier
| 32 | proTxHash | uint256 | The ProRegTx hash of the complaining member
| 1-9 | skContributions<br>Count | compactSize uint | Number of unencrypted secret key contributions
| 36 * `skContributions`<br>`Count` | skContribution | SKContribution | Member index and secret key contribution for members justifying complaints
| 96 | sig | byte[] | BLS signature, signed with the operator key of the contributing masternode

An `SKContribution` consists of:

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 4 | skContributionMember | uint32_t | Index of the member for which justification is provided
| 32 | skContributions | byte[] | Unencrypted secret key contribution for the member contained in skContributionMember

More information can be found in the [Justification phase section of DIP6](https://github.com/dashpay/dips/blob/master/dip-0006.md#4-justification-phase).

The following annotated hexdump shows a `qjustify` message. (The message header has been omitted.)

~~~
01 ......................................... LLMQ Type: 1 (LLMQ_50_60)

b34b2bcb3430f403663e37be9c63c88e
4ca1f12c41846064cf960a0800000000 ........... Quorum Hash

e7d909afba6848f3fdf98b2da31db07e
3fbee621d58c469dce96d6283bcd4b25 ........... ProRegTx hash

05 ......................................... Contribution count: 5

Contribution #1
| 16000000 ................................. Member Index: 22
|
| 57b63ec5ae0a101f0d93bb60af70bf22
| c21bd3a7705e1aecb9559d6b151d953f ......... Unencrypted secret key contribution

Contribution #2
| 17000000 ................................. Member Index: 22
|
| 0ee1f0f0f2570589e81d2a4f8165b105
| 28436a1a75cf3469fa81090f2d856150 ......... Unencrypted secret key contribution

[...] ...................................... 3 more contributions omitted

8d63d10e242ac97c6324e9a40d6e690e
4bb7fe0750b7d204f7e988a324720189
68408d2d0621bbaba8380ad4aaf342ea
138ce9a59ed9ca82995c155609488dcc
5ac35f483b695a0624e5ab0745f7f9e2
051edf1b3b1f0e1b1d55d185d25e0ed7 ........... BLS signature (Operator Key)
~~~


## qpcommit

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qpcommit` message is used to exchange premature commitment messages for verification and selection of the final commitment.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | llmqType | uint8_t | The type of LLMQ
| 32 | quorumHash | uint256 | The quorum identifier
| 32 | proTxHash | uint256 | The ProRegTx hash of the complaining member
| 1-9 | validMembersSize | compactSize uint | Bit size of the `validMembers` bitvector
| (`valid`<br>`MembersSize` + 7) / 8 | validMembers | byte[] | Bitset of valid members in this commitment
| 48 | quorumPublicKey | uint256 | The quorum public key
| 32 | quorumVvecHash | byte[] | The hash of the quorum verification vector
| 96 | quorumSig | BLSSig | Threshold signature, signed with the threshold signature share of the committing member
| 96 | sig | byte[] | BLS signature, signed with the operator key of the contributing masternode

More information can be found in the [Commitment phase section of DIP6](https://github.com/dashpay/dips/blob/master/dip-0006.md#5-commitment-phase).

The following annotated hexdump shows a `qpcommit` message. (The message header has been omitted.)

~~~
01 ......................................... LLMQ Type: 1 (LLMQ_50_60)

cb9a1552340175a8232437eb8ceceaea
4b90a0f75caff20ee12d230b00000000 ........... Quorum Hash

59c38b8d6a0664411f92a6326e8ef070
7ecf185405252854ddb477d89127a32d ........... ProRegTx hash

32 ......................................... Valid member bitvector size: 50
ffffffffffff03 ............................. Valid members

102809b8649209a15fceb3984014eb39
70ca9bd2464b2f84353a3353f4d612eb
7ca6daaf723170cdbdad40c5cf44f87b ........... Quorum BLS Public Key

17431ce7dfecb9bba4ccba5921514d24
fe267c61078bdfe29d90774a3b766ad5 ........... Quorum Verification Vector Hash

94f7417e0ed56ada7116cf4f1e400748
deb2e2040babd540f21925b2eec8d4df
75d3e0fc3323d083db76f66ce6128a13
0f1b2c4725076dae2283bbecbf2e1230
72cc9cec244337008bf82a670ab9e2ee
6220dd736a1a70c9ca87867ca55f8665 ........... BLS Threshold signature

85723fe503bba8ac814eab0f28f1fd07
49927528c01b635d11d3f2843ce3f7e1
6223c7e9a9e1f70916159c965acae8bf
09d16dc85267ec4081907adc966eae69
b6a5077267fdc61cdb192faffa27bed9
2883559bab2ab81cef6253452622b30c ........... BLS signature (Operator Key)
~~~


## qfcommit

The `qfcommit` message is used to finalize a long-living masternode quorum setup by aggregating the information necessary to mine the on-chain QcTx special transaction. The message contains all the necessary information required to validate the long-living masternode quorum's signing results.

It is possible to receive multiple valide final commitments for the same DKG session. These should only differ in the number of signers, which can be ignored as long as there are at least `quorumThreshold` number of signers. The set of valid members for these final commitments should always be the same, as each member only creates a single premature commitment. This means that only one set of valid members (and thus only one quorum verification vector and quorum public key) can gain a majority. If the threshold is not reached, there will be no valid final commitment.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 2 | version | uint16_t | Version of the final commitment message
| 1 | llmqType | uint8_t | The type of LLMQ
| 32 | quorumHash | uint256 | The quorum identifier
| 1-9 | signersSize | compactSize uint | Bit size of the signers bitvector
| (bitSize + 7) / 8 | signers | byte[] | Bitset representing the aggregated signers of this final commitment
| 1-9 | validMembersSize | compactSize uint | Bit size of the `validMembers` bitvector
| (bitSize + 7) / 8 | validMembers | byte[] | Bitset of valid members in this commitment
| 48 | quorumPublicKey | BLSPubKey | The quorum public key
| 32 | quorumVvecHash | uint256 | The hash of the quorum verification vector
| 96 | quorumSig | byte[] | Recovered threshold signature
| 96 | sig | byte[] | Aggregated BLS signatures from all included commitments

More information can be found in the [Finalization phase section of DIP6](https://github.com/dashpay/dips/blob/master/dip-0006.md#6-finalization-phase).

The following annotated hexdump shows a `qfcommit` message. (The message header has been omitted.)

~~~
0100 ....................................... Message Version: 1
01 ......................................... LLMQ Type: 1 (LLMQ_50_60)

cb9a1552340175a8232437eb8ceceaea
4b90a0f75caff20ee12d230b00000000 ........... Quorum Hash

32 ......................................... Signer bitvector size: 50
ffffffffffff03 ............................. Signers

32 ......................................... Valid member bitvector size: 50
ffffffffffff03 ............................. Valid members

102809b8649209a15fceb3984014eb39
70ca9bd2464b2f84353a3353f4d612eb
7ca6daaf723170cdbdad40c5cf44f87b ........... Quorum BLS Public Key

17431ce7dfecb9bba4ccba5921514d24
fe267c61078bdfe29d90774a3b766ad5 ........... Quorum Verification Vector Hash

083388b91a2f8f7f4ea35469f25ee16a
21b3e03b02936675897f74424d6de748
66b34dcc5861fd3f5f661ea1ed124a08
0b165f21b1f2db18c4c37c82f8a8d350
9a6f52a14c643dab71a4dced78ae9a42
dc982e89a92606df537b8918881e9c95 ........... Quorum BLS Recovered Threshold Sig

0d131c7062253671f9c8ebb39a9b0057
d78dc67e236b55086cbb0624c7f4abcc
0a26557bfad3092bd38ded4e3cca6c43
0dda2e73a99ca3d359631cb99a121c5e
92cea06ef4c03bb18ad9e90559104550
c8a042dc51aa58a26c134405fc3234ff ........... Quorum Aggregate BLS Sig
~~~


# Signing Sessions

The following network messages enable the long living masternode quorum (LLMQ) message signing sessions described in [DIP7](https://github.com/dashpay/dips/blob/master/dip-0007.md).

With the exception of the `qsendrecsigs` message and the `qsigrec` message, these messages are for intra-quorum communication only and are not propagated on the Dash network.

## qbsigs

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qbsigs` message is used to send batched signature shares in response to a `qgetsigs` message.

Note: The number of messages that can be sent in a batch is limited to 400 (as defined by `MAX_MSGS_TOTAL_BATCHED_SIGS` in Dash Core).

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| Varies | batchCount | compactSize uint | Number of batched signature shares |
| Varies | msgs | CBatchedSigShares | Batches of signature shares |

CBatchedSigShares:

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| Varies | sessionId | varint | Signing session ID |
| Varies | shareCount | compactSize uint | Number of shares |
| shareCount * 98 | sigShares | <uint16_t, CBLSLazySignature> | Index (2 bytes) and BLS Signature share (96 bytes) |

The following annotated hexdump shows a `qbsigs` message. (The message header has been omitted.)

~~~
02 ......................................... Number of signature share batches: 2

Signature Share Batch 1
| 84d843 ................................... Session ID
|
| 01 ....................................... Share count: 1
|
| Share
| | 2100 ................................... Index
|
| | 0fbd0c0981b79544c3e80d1a2eed13fe
| | f08c731b0156654675209812f9b2b8f3
| | ec23868d26890a0e85e5cec4ad0e2d46
| | 01293cf7e41841fda5865063e7354f36
| | e8a5c13d2c2d265a778f41e807b3cc63
| | 81e202ecf923c62bbb69ecc713bdf86d ....... BLS Signature share

Signature Share Batch 2
| 84d844 ................................... Session ID
|
| 01 ....................................... Share Count: 1
|
| Share
| | 2100 ................................... Index
| |
| | 9570d97e41b78045b51fba3d4f1ea38d
| | 7a0e007535ce6beb1e03eff163b421fd
| | b8125142a12f92aa82770de7bb038207
| | 13ccc72dd6d9bf91ecc2835da54a0afb
| | 0c0fa5d7a214a020ca650ca202ddff29
| | c3cac4033098297d2aaee098db5bfe2f ....... BLS Signature share
~~~


## qgetsigs

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qgetsigs` message is used to request signature shares. The response to a `qgetsigs` message is a `qbsigs` message.

Note: The number of inventories in a `qgetsigs` message is limited to 200 (as defined by `MAX_MSGS_CNT_QGETSIGSHARES` in Dash Core).

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| Varies | count | compactSize uint | Number of signature shares requested |
| Varies | sessionId | varint | Signing session ID
| Varies | invSize | compactSize uint | Inventory size
| Varies | inv | CAutoBitSet | Quorum signature inventory |

The following annotated hexdump shows a `qgetsigs` message. (The message header has been omitted.)

~~~
02 ......................................... Count: 2

Signature share request 1
| 80db21 ................................... Session ID
| 32 ....................................... Inventory size: 50
| 012900 ................................... Inventory

Signature share request 2
| 80db22 ................................... Session ID
| 32 ....................................... Inventory Size: 50
| 012900 ................................... Inventory
~~~


## qsendrecsigs

*Added in protocol version 70214 of Dash Core*

The `qsendrecsigs` message is used to notify a peer to send plain LLMQ recovered signatures (inventory type `MSG_QUORUM_RECOVERED_SIG`). Otherwise the peer would only announce/send the higher level messages produced when a recovered signature is found (e.g. InstantSend `islock` messages or ChainLock `clsig` messages).

Note: SPV nodes should not send this message as they are usually only interested in the higher level messages.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | fSendRecSigs | bool | 0 - Notify peer to not send plain LLMQ recovered signatures<br>1 - Notify peer to send plain LLMQ recovered signatures (default for Dash Core nodes)

The following annotated hexdump shows a `qsendrecsigs` message. (The message header has been omitted.)

~~~
01 ................................. Request recovered signatures: Enabled (1)
~~~


## qsigrec

*Added in protocol version 70214 of Dash Core*

The `qsigrec` message is used to provide recovered signatures and related quorum details to nodes that have requested this information via the `qsendrecsigs` message.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | llmqType | uint8_t | The type of LLMQ
| 32 | quorumHash | uint256 | The quorum hash
| 32 | id | uint256 | The signing request id
| 32 | msgHash | uint256 | The message hash
| 96 | sig | byte[] | The final recovered BLS threshold signature

More information can be found in the [Recovered threshold signatures section of DIP7](https://github.com/dashpay/dips/blob/master/dip-0007.md#recovered-threshold-signatures).

The following annotated hexdump shows a `qsigrec` message. (The message header has been omitted.)

Note: The following `qsigrec` message corresponds to the example `islock` message hexdump. The message hash below corresponds to the `islock` TXID field and the BLS signature matches the BLS signature of the `islock` example.

~~~
01 ......................................... LLMQ Type: 1 (LLMQ_50_60)

7d0befca14fa9e594aa19deab138ef28
23fe838c89ed9be6ddc63c0200000000 ........... Quorum Hash

0f1937c60f35640d063eae8eb288af21
a2ec0ec69b58b20c52f5d438eaabd54d ........... Signing Request ID

e2e1c797576d8b13c83e929684b9aacd
553c20a34e2d11e38bdcaaf8e1de1680 ........... Message Hash

0f055c2064885d446f83d51b9bb09892
7ea0375a0f6a3f3402abf158ece67e00
81049b8a8f45d254b64574cef194ef31
197e450fba1196d652f2e1421d3b80ae
f429c10eabd4ab9289e9a8f80f6989b7
a11e5e7930deccc3e11a931fc9524f06 ........... LLMQ BLS Signature (96 bytes)
~~~


## qsigsesann

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qsigsesann` message is used to announce the sessionId for a signing session. The sessionId will be used for all P2P messages related to that session.

Note: The maximum number of announcements in a `qsigsesann` message is limited to 100 (as defined by `MAX_MSGS_CNT_QSIGSESANN` in Dash Core).

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| Varies | count | compactSize uint | Number of session announcements |
| Varies | sessionId | varint | Signing session ID (must be less than the maximum uint32_t value)
| 1 | llmqType | uint8_t | The LLMQ type
| 32 | quorumHash | uint256 | The quorum identifier
| 32 | id | uint256 | The signing request id
| 32 | msgHash | uint256 | The message hash

The following annotated hexdump shows a `qsigsesann` message. (The message header has been omitted.)

~~~
02 ......................................... Count: 2

Session Announcement 1
| 84d843 ................................... Session ID
|
| 01 ....................................... LLMQ Type: 1 (LLMQ_50_60)
|
| a34d3ae6b33cb1199c3e5e1cb5a2a55c
| 91e69bb5df2bf80ba1cb0a0d00000000 ......... Quorum Hash
|
| 89bbc2e5741a9f706e8d33dee4132037
| 8c33511768c5e3d6cdb2a1b7b731360b ......... Signing request ID
|
| d2b41a19237e370b4b091545b203bc0c
| 02ca7e0d5daebf12bb24b13064ed4149 ......... Message Hash

Session Announcement 2
| 84d844 ................................... Session ID
|
| 01 ....................................... LLMQ Type: 1 (LLMQ_50_60)
|
| a34d3ae6b33cb1199c3e5e1cb5a2a55c
| 91e69bb5df2bf80ba1cb0a0d00000000 ......... Quorum Hash
|
| 54f73deb42a8ed9b72b9c0535a72f54d
| 5789bbe0dbea2e184c3089f9e8f65c3e ......... Signing request ID
|
| af2e5d730cd37cd911b92db117b4ab99
| 90a3c0300ce39177d0d31be5b47c2361 ......... Message Hash
~~~


## qsigsinv

*Added in protocol version 70214 of Dash Core*

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) Note: This message is used for intra-quorum communication and is only sent to the masternodes in the LLMQ and nodes that are monitoring in Watch Mode for auditing/debugging purposes.

The `qsigsinv` message (quorum signature inventory) announces one or more quorum signature share inventories known by the transmitting peer.

Note: The maximum number of inventories in a `qsigsinv` message is limited to 200 (as defined by `MAX_MSGS_CNT_QSIGSHARESINV` in Dash Core).

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| Varies | count | compactSize uint | Number of session announcements |
| Varies | sessionId | varint | Signing session ID (must be less than the maximum uint32_t value) |
| Varies | invSize | compactSize uint | Inventory size
| Varies | inv | CAutoBitSet | Quorum signature inventory |

The following annotated hexdump shows a `qsigsinv` message. (The message header has been omitted.)

~~~
02 ......................................... Count: 2

84d844 ..................................... Session ID
32 ......................................... Inventory size: 50
011a040200 ................................. Inventory

84d843 ..................................... Session ID
32 ......................................... Inventory size: 50
011a0700 ................................... Inventory
~~~

# Debugging

## qwatch

*Added in protocol version 70214 of Dash Core*

The `qwatch` message tells the receiving peer to relay LLMQ messages (`qcontrib` messages, `qcomplaint` messages, `qjustify` messages, and `qpcommit` messages).

There is no payload in a `qwatch` message.  See the [message header section][section message header] for an example of a message without a payload.