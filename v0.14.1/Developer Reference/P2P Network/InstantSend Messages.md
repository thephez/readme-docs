---
title: "InstantSend Messages"
excerpt: ""
---
The following network messages all help control the InstantSend feature of Dash. InstantSend uses the masternode network to lock transaction inputs and enable secure, instantaneous transactions. For additional details, refer to the Developer Guide [InstantSend section](#guide-instantsend).

![Overview Of P2P Protocol InstantSend Request And Reply Messages](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-p2p-instantsend-messages.png)


# clsig

*Added in protocol version 70214 of Dash Core*

The `clsig` message is used to indicate a successful ChainLock for the designated block height. The Chainlock ensures that no other blocks can replace the one with the indicated block hash. This determination is made by agreement of a long-living masternode quorum (LLMQ) which creates the BLS signature in the message.

Once a `clsig` message is received, clients must reject any other blocks for the indicated block height as described in [DIP8 (ChainLocks)](https://github.com/dashpay/dips/blob/master/dip-0008.md). This increases security by preventing reorganization of a block with a ChainLock (and all blocks below it).

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 4 | nHeight | int32_t | Block height
| 32 | blockHash | uint256 | Block hash
| 96 | sig | CBLSSignature | LLMQ BLS signature

The following annotated hexdump shows a `clsig` message. (The message header has been omitted.)

~~~
c1310100 ................................... Block Height: 78273

03bb286a877135fad3b33ea9cce9a525
e5edc0879411afaff513b30100000000 ........... Block Hash

12a3331fd8d0008804edaaf94c57b491
d36f38f1993d06dfff71df9ec83da463
dcd5497d105932e609016dac075f02df
1259951e3bcebfcc26afc904f6cd92df
7ff9b8c6c2ac9dcc9cb1a7dc7ec03bcc
005574710c3aabc2f8670959cf8bc9b5 ........... LLMQ BLS Signature
~~~

# islock

*Added in protocol version 70214 of Dash Core*

The `islock` message is used to provide details of transactions that have been locked by LLMQ-based InstantSend. The message includes the details of all transaction inputs along with the transaction ID and the BLS signature of the LLMQ that approved the transaction lock.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1-9 | inputsSize | compactSize uint | Number of inputs |
| 36 * `inputsSize`| inputs | COutPoint | Outpoints used in the transaction |
| 32 | txid | uint256 | TXID of the locked transaction |
| 96 | sig | byte[] | LLMQ BLS Signature |

The following annotated hexdump shows a `islock` message. (The message header has been omitted.)

~~~
02 ......................................... Number of inputs: 2

Input 1
| 05f24f65a9d98ff543ba61f7f0ce9448
| 79632bf2517540a5bd8bde2fe8e94617 ......... Previous outpoint TXID
| 00000000 ................................. Previous outpoint index: 0

Input 2
| 05f24f65a9d98ff543ba61f7f0ce9448
| 79632bf2517540a5bd8bde2fe8e94617 ......... Previous outpoint TXID
| 01000000 ................................. Previous outpoint index: 1

e2e1c797576d8b13c83e929684b9aacd
553c20a34e2d11e38bdcaaf8e1de1680 ........... TXID

0f055c2064885d446f83d51b9bb09892
7ea0375a0f6a3f3402abf158ece67e00
81049b8a8f45d254b64574cef194ef31
197e450fba1196d652f2e1421d3b80ae
f429c10eabd4ab9289e9a8f80f6989b7
a11e5e7930deccc3e11a931fc9524f06 ........... LLMQ BLS Signature (96 bytes)
~~~