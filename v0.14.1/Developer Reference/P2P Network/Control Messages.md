---
title: "Control Messages"
excerpt: ""
---
The following network messages all help control the connection between two peers or allow them to advise each other about the rest of the network.

![Overview Of P2P Protocol Control And Advisory Messages](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-p2p-control-messages.png)

Note that almost none of the control messages are authenticated in any way, meaning they can contain incorrect or intentionally harmful information.

# Addr

The `addr` (IP address) message relays connection information for peers on the network. Each peer which wants to accept incoming connections creates an `addr` message providing its connection information and then sends that message to its peers unsolicited. Some of its peers send that information to their peers (also unsolicited), some of which further distribute it, allowing decentralized peer discovery for any program already on the network.

An `addr` message may also be sent in response to a `getaddr` message.

| Bytes      | Name             | Data Type          | Description
|------------|------------------|--------------------|----------------
| *Varies*   | IP address count | compactSize uint   | The number of IP address entries up to a maximum of 1,000.
| *Varies*   | IP addresses     | network IP address | IP address entries.  See the table below for the format of a Dash network IP address.

Each encapsulated network IP address currently uses the following structure:

| Bytes | Name       | Data Type | Description
|-------|------------|-----------|---------------
| 4     | time       | uint32    | *Added in protocol version 31402.* <br><br>A time in Unix epoch time format.  Nodes advertising their own IP address set this to the current time.  Nodes advertising IP addresses they've connected to set this to the last time they connected to that node.  Other nodes just relaying the IP address should not change the time.  Nodes can use the time field to avoid relaying old `addr` messages.  <br><br>Malicious nodes may change times or even set them in the future.
| 8     | services   | uint64_t  | The services the node advertised in its `version` message.
| 16    | IP address | char      | IPv6 address in **big endian byte order**. IPv4 addresses can be provided as [IPv4-mapped IPv6 addresses][]
| 2     | port       | uint16_t  | Port number in **big endian byte order**.  Note that Dash Core will only connect to nodes with non-standard port numbers as a last resort for finding peers.  This is to prevent anyone from trying to use the network to disrupt non-Dash services that run on other ports.

The following annotated hexdump shows part of an `addr` message. (The message header has been omitted and the actual IP address has been replaced with a [RFC5737][] reserved IP address.)

~~~
fde803 ............................. Address count: 1000

d91f4854 ........................... Epoch time: 1414012889
0100000000000000 ................... Service bits: 01 (network node)
00000000000000000000ffffc0000233 ... IP Address: ::ffff:192.0.2.51
208d ............................... Port: 8333

[...] .............................. (999 more addresses omitted)
~~~


# FilterAdd

*Added in protocol version 70001 as described by BIP37.*

The `filteradd` message tells the receiving peer to add a single element to a previously-set bloom filter, such as a new public key. The element is sent directly to the receiving peer; the peer then uses the parameters set in the `filterload` message to add the element to the bloom filter.

Because the element is sent directly to the receiving peer, there is no obfuscation of the element and none of the plausible-deniability privacy provided by the bloom filter. Clients that want to maintain greater privacy should recalculate the bloom filter themselves and send a new `filterload` message with the recalculated bloom filter.

| Bytes    | Name          | Data Type        | Description
|----------|---------------|------------------|-----------------
| *Varies* | element bytes | compactSize uint | The number of bytes in the following element field.
| *Varies* | element       | uint8_t[]        | The element to add to the current filter.  Maximum of 520 bytes, which is the maximum size of an element which can be pushed onto the stack in a pubkey or signature script.  Elements must be sent in the byte order they would use when appearing in a raw transaction; for example, hashes should be sent in internal byte order.

Note: a `filteradd` message will not be accepted unless a filter was previously set with the `filterload` message.

The annotated hexdump below shows a `filteradd` message adding a TXID. (The message header has been omitted.) This TXID appears in the same block used for the example hexdump in the `merkleblock` message; if that `merkleblock` message is re-sent after sending this `filteradd` message, six hashes are returned instead of four.

~~~
20 ................................. Element bytes: 32
fdacf9b3eb077412e7a968d2e4f11b9a
9dee312d666187ed77ee7d26af16cb0b ... Element (A TXID)
~~~


# FilterClear

*Added in protocol version 70001 as described by BIP37.*

The `filterclear` message tells the receiving peer to remove a previously-set bloom filter.  This also undoes the effect of setting the relay field in the `version` message to 0, allowing unfiltered access to `inv` messages announcing new transactions.

Dash Core does not require a `filterclear` message before a replacement filter is loaded with `filterload`.  It also doesn't require a `filterload` message before a `filterclear` message.

There is no payload in a `filterclear` message.  See the [message header section][section message header] for an example of a message without a payload.


# FilterLoad

*Added in protocol version 70001 as described by BIP37.*

The `filterload` message tells the receiving peer to filter all relayed transactions and requested merkle blocks through the provided filter. This allows clients to receive transactions relevant to their wallet plus a configurable rate of false positive transactions which can provide plausible-deniability privacy.

| Bytes    | Name         | Data Type | Description
|----------|--------------|-----------|---------------
| *Varies* | nFilterBytes | compactSize uint | Number of bytes in the following filter bit field.
| *Varies* | filter       | uint8_t[] | A bit field of arbitrary byte-aligned size. The maximum size is 36,000 bytes.
| 4        | nHashFuncs   | uint32_t  | The number of hash functions to use in this filter. The maximum value allowed in this field is 50.
| 4        | nTweak       | uint32_t  | An arbitrary value to add to the seed value in the hash function used by the bloom filter.
| 1        | nFlags       | uint8_t   | A set of flags that control how outpoints corresponding to a matched pubkey script are added to the filter. See the table in the Updating A Bloom Filter subsection below.

The annotated hexdump below shows a `filterload` message. (The message header has been omitted.)  For an example of how this payload was created, see the [filterload example][section creating a bloom filter].

~~~
02 ......... Filter bytes: 2
b50f ....... Filter: 1010 1101 1111 0000
0b000000 ... nHashFuncs: 11
00000000 ... nTweak: 0/none
00 ......... nFlags: BLOOM_UPDATE_NONE
~~~

**Initializing A Bloom Filter**

Filters have two core parameters: the size of the bit field and the number of hash functions to run against each data element. The following formulas from BIP37 will allow you to automatically select appropriate values based on the number of elements you plan to insert into the filter (*n*) and the false positive rate (*p*) you desire to maintain plausible deniability.

* Size of the bit field in bytes (*nFilterBytes*), up to a maximum of
  36,000: `(-1 / log(2)**2 * n * log(p)) / 8`

* Hash functions to use (*nHashFuncs*), up to a maximum of 50:
  `nFilterBytes * 8 / n * log(2)`

Note that the filter matches parts of transactions (transaction elements), so the false positive rate is relative to the number of elements checked---not the number of transactions checked. Each normal transaction has a minimum of four matchable elements (described in the comparison subsection below), so a filter with a false-positive rate of 1 percent will match about 4 percent of all transactions at a minimum.

According to BIP37, the formulas and limits described above provide support for bloom filters containing 20,000 items with a false positive rate of less than 0.1 percent or 10,000 items with a false positive rate of less than 0.0001 percent.

Once the size of the bit field is known, the bit field should be initialized as all zeroes.

**Populating A Bloom Filter**

The bloom filter is populated using between 1 and 50 unique hash functions (the number specified per filter by the *nHashFuncs* field). Instead of using up to 50 different hash function implementations, a single implementation is used with a unique seed value for each function.

The seed is `nHashNum * 0xfba4c795 + nTweak` as a *uint32\_t*, where the values are:

* **nHashNum** is the sequence number for this hash
  function, starting at 0 for the first hash iteration and increasing up
  to the value of the *nHashFuncs* field (minus one) for the last hash
  iteration.

* **0xfba4c795** is a constant optimized to create large differences in
  the seed for different values of *nHashNum*.

* **nTweak** is a per-filter constant set by the client to require the use
  of an arbitrary set of hash functions.

If the seed resulting from the formula above is larger than four bytes, it must be truncated to its four most significant bytes (for example, `0x8967452301 & 0xffffffff → 0x67452301`).

The actual hash function implementation used is the [32-bit Murmur3 hash function][murmur3].

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) **Warning:** the Murmur3 hash function has separate 32-bit and 64-bit versions that produce different results for the same input.  Only the 32-bit Murmur3 version is used with Dash bloom filters.

The data to be hashed can be any transaction element which the bloom filter can match. See the next subsection for the list of transaction elements checked against the filter. The largest element which can be matched is a script data push of 520 bytes, so the data should never exceed 520 bytes.

The example below from Dash Core [bloom.cpp][core bloom.cpp hash] combines all the steps above to create the hash function template. The seed is the first parameter; the data to be hashed is the second parameter. The result is a uint32\_t modulo the size of the bit field in bits.

```c
MurmurHash3(nHashNum * 0xFBA4C795 + nTweak, vDataToHash) % (vData.size() * 8)
```

Each data element to be added to the filter is hashed by *nHashFuncs* number of hash functions. Each time a hash function is run, the result will be the index number (*nIndex*) of a bit in the bit field. That bit must be set to 1. For example if the filter bit field was `00000000` and the result is 5, the revised filter bit field is `00000100` (the first bit is bit 0).

It is expected that sometimes the same index number will be returned more than once when populating the bit field; this does not affect the algorithm---after a bit is set to 1, it is never changed back to 0.

After all data elements have been added to the filter, each set of eight bits is converted into a little-endian byte. These bytes are the value of the *filter* field.

**Comparing Transaction Elements To A Bloom Filter**

To compare an arbitrary data element against the bloom filter, it is hashed using the same parameters used to create the bloom filter. Specifically, it is hashed *nHashFuncs* times, each time using the same *nTweak* provided in the filter, and the resulting output is modulo the size of the bit field provided in the *filter* field.  After each hash is performed, the filter is checked to see if the bit at that indexed location is set.  For example if the result of a hash is `5` and the filter is `01001110`, the bit is considered set.

If the result of every hash points to a set bit, the filter matches. If any of the results points to an unset bit, the filter does not match.

The following transaction elements are compared against bloom filters. All elements will be hashed in the byte order used in blocks (for example, TXIDs will be in internal byte order).

* **TXIDs:** the transaction's SHA256(SHA256()) hash.

* **Outpoints:** each 36-byte outpoint used this transaction's input
  section is individually compared to the filter.

* **Signature Script Data:** each element pushed onto the stack by a
  data-pushing opcode in a signature script from this transaction is
  individually compared to the filter.  This includes data elements
  present in P2SH redeem scripts when they are being spent.

* **PubKey Script Data:** each element pushed onto the the stack by a
  data-pushing opcode in any pubkey script from this transaction is
  individually compared to the filter. (If a pubkey script element
  matches the filter, the filter will be immediately updated if the
  `BLOOM_UPDATE_ALL` flag was set; if the pubkey script is in the P2PKH
  format and matches the filter, the filter will be immediately updated
  if the `BLOOM_UPDATE_P2PUBKEY_ONLY` flag was set. See the subsection
  below for details.)

As of  Dash Core 0.14.0, elements in the extra payload section of DIP2-based special transactions are also compared against bloom filters.

The following annotated hexdump of a transaction is from the [raw transaction format section][raw transaction format]; the elements which would be checked by the filter are emphasized in bold. Note that this transaction's TXID (**`01000000017b1eab[...]`**) would also be checked, and that the outpoint TXID and index number below would be checked as a single 36-byte element.

~~~
01000000 ................................... Version

01 ......................................... Number of inputs
|
| <b>7b1eabe0209b1fe794124575ef807057</b>
| <b>c77ada2138ae4fa8d6c4de0398a14f3f</b> ......... Outpoint TXID
| <b>00000000</b> ................................. Outpoint index number
|
| 49 ....................................... Bytes in sig. script: 73
| | 48 ..................................... Push 72 bytes as data
| | | <b>30450221008949f0cb400094ad2b5eb3</b>
| | | <b>99d59d01c14d73d8fe6e96df1a7150de</b>
| | | <b>b388ab8935022079656090d7f6bac4c9</b>
| | | <b>a94e0aad311a4268e082a725f8aeae05</b>
| | | <b>73fb12ff866a5f01</b> ..................... Secp256k1 signature
|
| ffffffff ................................. Sequence number: UINT32_MAX

01 ......................................... Number of outputs
| f0ca052a01000000 ......................... Satoshis (49.99990000 BTC)
|
| 19 ....................................... Bytes in pubkey script: 25
| | 76 ..................................... OP_DUP
| | a9 ..................................... OP_HASH160
| | 14 ..................................... Push 20 bytes as data
| | | <b>cbc20a7664f2f69e5355aa427045bc15</b>
| | | <b>e7c6c772</b> ............................. PubKey hash
| | 88 ..................................... OP_EQUALVERIFY
| | ac ..................................... OP_CHECKSIG

00000000 ................................... locktime: 0 (a block height)
~~~

**Updating A Bloom Filter**

Clients will often want to track inputs that spend outputs (outpoints) relevant to their wallet, so the filterload field *nFlags* can be set to allow the filtering node to update the filter when a match is found. When the filtering node sees a pubkey script that pays a pubkey, address, or other data element matching the filter, the filtering node immediately updates the filter with the outpoint corresponding to that pubkey script.

![Automatically Updating Bloom Filters](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-bloom-update.png)

If an input later spends that outpoint, the filter will match it, allowing the filtering node to tell the client that one of its transaction outputs has been spent.

The *nFlags* field has three allowed values:

| Value | Name                       | Description
|-------|----------------------------|---------------
| 0     | BLOOM_UPDATE_NONE          | The filtering node should not update the filter.
| 1     | BLOOM_UPDATE_ALL           | If the filter matches any data element in a pubkey script, the corresponding outpoint is added to the filter.
| 2     | BLOOM_UPDATE_P2PUBKEY_ONLY | If the filter matches any data element in a pubkey script and that script is either a P2PKH or non-P2SH pay-to-multisig script, the corresponding outpoint is added to the filter.

In addition, because the filter size stays the same even though additional elements are being added to it, the false positive rate increases. Each false positive can result in another element being added to the filter, creating a feedback loop that can (after a certain point) make the filter useless. For this reason, clients using automatic filter updates need to monitor the actual false positive rate and send a new filter when the rate gets too high.


# GetAddr

The `getaddr` message requests an `addr` message from the receiving node, preferably one with lots of IP addresses of other receiving nodes. The transmitting node can use those IP addresses to quickly update its database of available nodes rather than waiting for unsolicited `addr` messages to arrive over time.

There is no payload in a `getaddr` message.  See the [message header section][section message header] for an example of a message without a payload.


# GetSporks

The `getsporks` message requests `spork` messages from the receiving node.

There is no payload in a `getsporks` message.  See the [message header section][section message header] for an example of a message without a payload.


# Ping

The `ping` message helps confirm that the receiving peer is still connected. If a TCP/IP error is encountered when sending the `ping` message (such as a connection timeout), the transmitting node can assume that the receiving node is disconnected. The response to a `ping` message is the `pong` message.

Before protocol version 60000, the `ping` message had no payload. As of protocol version 60001 and all later versions, the message includes a single field, the nonce.

| Bytes | Name  | Data Type | Description
|-------|-------|-----------|---------------
| 8     | nonce | uint64_t  | *Added in protocol version 60001 as described by BIP31.* <br><br>Random nonce assigned to this `ping` message.  The responding `pong` message will include this nonce to identify the `ping` message to which it is replying.

The annotated hexdump below shows a `ping` message. (The message header has been omitted.)

~~~
0094102111e2af4d ... Nonce
~~~


# Pong

*Added in protocol version 60001 as described by BIP31.*

The `pong` message replies to a `ping` message, proving to the pinging node that the ponging node is still alive. Dash Core will, by default, disconnect from any clients which have not responded to a `ping` message within 20 minutes.

To allow nodes to keep track of latency, the `pong` message sends back the same nonce received in the `ping` message it is replying to.

The format of the `pong` message is identical to the `ping` message; only the message header differs.


# Reject

*Added in protocol version 70002 as described by BIP61.*

The `reject` message informs the receiving node that one of its previous messages has been rejected.

| Bytes    | Name          | Data Type        | Description
|----------|---------------|------------------|--------------
| *Varies* | message bytes | compactSize uint | The number of bytes in the following message field.
| *Varies* | message       | string           | The type of message rejected as ASCII text *without null padding*.  For example: "tx", "block", or "version".
| 1        | code          | char             | The reject message code.  See the table below.
| *Varies* | reason bytes  | compactSize uint | The number of bytes in the following reason field.  May be 0x00 if a text reason isn't provided.
| *Varies* | reason        | string           | The reason for the rejection in ASCII text.  This should not be displayed to the user; it is only for debugging purposes.
| *Varies* | extra data    | *varies*         | Optional additional data provided with the rejection.  For example, most rejections of `tx` messages or `block` messages include the hash of the rejected transaction or block header.  See the code table below.

The following table lists message reject codes.  Codes are tied to the type of message they reply to; for example there is a 0x10 reject code for transactions and a 0x10 reject code for blocks.

| Code | In Reply To       | Extra Bytes | Extra Type | Description
|------|-------------------|-------------|------------|----------------
| 0x01 | *any message*     | 0           | N/A        | Message could not be decoded.  Be careful of `reject` message feedback loops where two peers each don't understand each other's `reject` messages and so keep sending them back and forth forever.
| 0x10 | `block` message   | 32          | char[32]   | Block is invalid for some reason (invalid proof-of-work, invalid signature, etc).  Extra data may include the rejected block's header hash.
| 0x10 | `tx` message      | 32          | char[32]   | Transaction is invalid for some reason (invalid signature, output value greater than input, etc.).  Extra data may include the rejected transaction's TXID.
| 0x10 | `ix` message      | 32          | char[32]   | InstantSend transaction is invalid for some reason (invalid tx lock request, conflicting tx lock request, etc.).  Extra data may include the rejected transaction's TXID.
| 0x11 | `block` message   | 32          | char[32]   | The block uses a version that is no longer supported.  Extra data may include the rejected block's header hash.
| 0x11 | `version` message | 0           | N/A        | Connecting node is using a protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `dsa` message     | 0           | N/A        | Connecting node is using a PrivateSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `dsi` message     | 0           | N/A        | Connecting node is using a PrivateSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `dsc` message     | 0           | N/A        | Connecting node is using a PrivateSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `dsf` message     | 0           | N/A        | Connecting node is using a PrivateSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `dsq` message     | 0           | N/A        | Connecting node is using a PrivateSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `dssu` message    | 0           | N/A        | Connecting node is using a PrivateSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `govsync` message | 0           | N/A        | Connecting node is using a governance protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `govobj` message  | 0           | N/A        | Connecting node is using a governance protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `govobjvote` message | 0           | N/A        | Connecting node is using a governance protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `mnget` message   | 0           | N/A        | Connecting node is using a masternode payment protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `mnw` message     | 0           | N/A        | Connecting node is using a masternode payment protocol version that the rejecting node considers obsolete and unsupported.
| 0x11 | `txlvote` message | 0           | N/A        | Connecting node is using an InstantSend protocol version that the rejecting node considers obsolete and unsupported.
| 0x12 | `tx` message      | 32          | char[32]   | Duplicate input spend (double spend): the rejected transaction spends the same input as a previously-received transaction.  Extra data may include the rejected transaction's TXID.
| 0x12 | `version` message | 0           | N/A        | More than one `version` message received in this connection.
| 0x40 | `tx` message      | 32          | char[32]   | The transaction will not be mined or relayed because the rejecting node considers it non-standard---a transaction type or version unknown by the server.  Extra data may include the rejected transaction's TXID.
| 0x41 | `tx` message      | 32          | char[32]   | One or more output amounts are below the dust threshold.  Extra data may include the rejected transaction's TXID.
| 0x42 | `tx` message      |             | char[32]   | The transaction did not have a large enough fee or priority to be relayed or mined.  Extra data may include the rejected transaction's TXID.
| 0x43 | `block` message   | 32          | char[32]   | The block belongs to a block chain which is not the same block chain as provided by a compiled-in checkpoint.  Extra data may include the rejected block's header hash.

Reject Codes

| Code | Description
|------|--------------
| 0x01 | Malformed
| 0x10 | Invalid
| 0x11 | Obsolete
| 0x12 | Duplicate
| 0x40 | Non-standard
| 0x41 | Dust
| 0x42 | Insufficient fee
| 0x43 | Checkpoint

The annotated hexdump below shows a `reject` message. (The message header has been omitted.)

~~~
02 ................................. Number of bytes in message: 2
7478 ............................... Type of message rejected: tx
12 ................................. Reject code: 0x12 (duplicate)
15 ................................. Number of bytes in reason: 21
6261642d74786e732d696e707574732d
7370656e74 ......................... Reason: bad-txns-inputs-spent
394715fcab51093be7bfca5a31005972
947baf86a31017939575fb2354222821 ... TXID
~~~


# SendCmpct

*Added in protocol version 70209 of Dash Core as described by BIP152*

The `sendcmpct` message tells the receiving peer whether or not to announce new blocks using a `cmpctblock` message. It also sends the compact block protocol version it supports. The `sendcmpct` message is defined as a message containing a 1-byte integer followed by a 8-byte integer. The first integer is interpreted as a boolean and should have a value of either 1 or 0. The second integer is be interpreted as a little-endian version number.

Upon receipt of a `sendcmpct` message with the first and second integers set to 1, the node should announce new blocks by sending a `cmpctblock` message.

Upon receipt of a `sendcmpct` message with the first integer set to 0, the node shouldn't announce new blocks by sending a `cmpctblock` message, but instead announce new blocks by sending invs or headers, as defined by [BIP130][].

Upon receipt of a `sendcmpct` message with the second integer set to something other than 1, nodes should treat the peer as if they had not received the message (as it indicates the peer will provide an unexpected encoding in `cmpctblock` messages, and/or other, messages). This allows future versions to send duplicate `sendcmpct` messages with different versions as a part of a version handshake.

Nodes should check for a protocol version of >= 70209 before sending `sendcmpct` messages. Nodes shouldn't send a request for a `MSG_CMPCT_BLOCK` object to a peer before having received a `sendcmpct` message from that peer. Nodes shouldn't request a `MSG_CMPCT_BLOCK` object before having sent all `sendcmpct` messages to that peer which they intend to send, as the peer cannot know what protocol version to use in the response.

The structure of a `sendcmpct` message is defined below.

| Bytes    | Name          | Data Type        | Description
|----------|---------------|------------------|--------------
| 1        | announce      | bool             | 0 - Announce blocks via `headers` message or `inv` message<br>1 - Announce blocks via `cmpctblock` message
| 8        | version       | uint64_t         | The compact block protocol version number

The annotated hexdump below shows a `sendcmpct` message. (The message header has been omitted.)

~~~
01 ................................. Block announce type: Compact Blocks
0100000000000000 ................... Compact block version: 1
~~~


# SendDsq

*Added in protocol version 70214 of Dash Core*

The `senddsq` message is used to notify a peer whether or not to send `dsq` messages. This allows clients that are not interested in PrivateSend mixing (e.g. mobile wallet) to minimize data usage.

| Bytes | Name | Data type | Description |
| --- | --- | --- | --- |
| 1 | fSendDSQueue | bool | 0 - Notify peer to not send any `dsq` messages<br>1 - Notify peer to send all `dsq` messages

The following annotated hexdump shows a `senddsq` message. (The message header has been omitted.)

~~~
01 ................................. PrivateSend participation: Enabled (1)
~~~


# SendHeaders

The `sendheaders` message tells the receiving peer to send new block announcements using a `headers` message rather than an `inv` message.

There is no payload in a `sendheaders` message.  See the [message header section][section message header] for an example of a message without a payload.


# Spork

Sporks are a mechanism by which updated code is released to the network, but not immediately made active (or “enforced”). Enforcement of the updated code can be activated remotely. Should problems arise, the code can be deactivated in the same manner, without the need for a network-wide rollback or client update.

A `spork` message may be sent in response to a `getsporks` message.

The `spork` message tells the receiving peer the status of the spork defined by the SporkID field. Upon receiving a spork message, the client must verify the signature before accepting the spork message as valid.

| Bytes | Name | Data type | Required | Description |
| ---------- | ----------- | --------- | -------- | -------- |
| 4 | nSporkID | int | Required | ID assigned in spork.h
| 8 | nValue | int64_t | Required | Value assigned to spork
| 8 | nTimeSigned | int64_t | Required | Time the spork value was signed
| 66 | vchSig | char[] | Required | Length (1 byte) + Signature (65 bytes)

Sporks (per [`src/spork.h`][spork.h])

| Spork ID | Number | Name | Description |
| ---------- | ---------- | ----------- | ----------- |
| 10001 | 2 | `INSTANTSEND_ENABLED` | Turns on and off InstantSend network wide
| 10002 | 3 | `INSTANTSEND_BLOCK_FILTERING` | Turns on and off InstantSend block filtering
| 10005 | 6 | `NEW_SIGS` | Turns on and off new signature format for Dash-specific messages
| 10008 | 9 | `SUPERBLOCKS_ENABLED` | Superblocks are enabled (10% of the block reward allocated to fund the dash treasury for funding approved proposals)
| 10014 | 15 | `DETERMINISTIC_MNS_ENABLED` | Deterministic masternode lists are enabled
| 10015 | 16 | `INSTANTSEND_AUTOLOCKS` | Automatic InstantSend for transactions with <=4 inputs (also eliminates the special InstantSend fee requirement for these transactions)
| 10016 | 17 | `SPORK_17_QUORUM_DKG_ENABLED` | Enable long-living masternode quorum (LLMQ) distributed key generation (DKG). When enabled, simple PoSe  scoring and banning is active as well.
| 10018 | 19 | `SPORK_19_CHAINLOCKS_ENABLED` | Enable LLMQ-based ChainLocks.
| 10019 | 20 | `SPORK_20_INSTANTSEND_LLMQ_BASED` | Enable LLMQ-based InstantSend.
| | | |
| | | **Removed Sporks** |
| _10004_ | _5_ | _`INSTANTSEND_MAX_VALUE`_ | _Removed in Dash Core 0.14.1.<br>Controls the max value for an InstantSend transaction (currently 2000 dash)_
| _10007_ | _8_ | _`MASTERNODE_PAYMENT_ENFORCEMENT`_ | _Removed in Dash Core 0.14.0.<br>Requires masternodes to be paid by miners when blocks are processed_
| _10009_ | _10_ | _`MASTERNODE_PAY_UPDATED_NODES`_ | _Removed in Dash Core 0.14.0.<br>Only current protocol version masternode's will be paid (not older nodes)_
| _10011_ | _12_ | _`RECONSIDER_BLOCKS`_ | _Removed in Dash Core 0.14.1.<br>Forces reindex of a specified number of blocks to recover from unintentional network forks_
| _10012_ | _13_ | _`OLD_SUPERBLOCK_FLAG`_ | _Removed in Dash Core 0.12.3.<br>No network function since block 614820_
| _10013_ | _14_ | _`REQUIRE_SENTINEL_FLAG`_ | _Removed in Dash Core 0.14.0.<br>Only masternode's running sentinel will be paid_
| _10017_ | _18_ | _`QUORUM_DEBUG_ENABLED`_ | _Removed in Dash Core 0.14.0.<br><br>Temporarily used on Testnet only quorum debugging._

To verify `vchSig`, compare the hard-coded spork public key (`strSporkPubKey` from [`src/chainparams.cpp`][spork pubkey]) with the public key recovered from the `spork` message's hash and `vchSig` value (implementation details for Dash Core can be found in `CPubKey::RecoverCompact`). The hash is a double SHA-256 hash of:

* The spork magic message (`"DarkCoin Signed Message:\n"`)
* nSporkID + nValue + nTimeSigned

| Network | Spork Pubkey (wrapped) |
| ---------- | ---------- |
| Mainnet | 04549ac134f694c0243f503e8c8a9a986f5de6610049c40b07816809b0d1<br>d06a21b07be27b9bb555931773f62ba6cf35a25fd52f694d4e1106ccd237<br>a7bb899fdd |
| Testnet3 | 046f78dcf911fbd61910136f7f0f8d90578f68d0b3ac973b5040fb7afb50<br>1b5939f39b108b0569dca71488f5bbf498d92e4d1194f6f941307ffd95f7<br>5e76869f0e |
| RegTest | Undefined |
| Devnets | 046f78dcf911fbd61910136f7f0f8d90578f68d0b3ac973b5040fb7afb50<br>1b5939f39b108b0569dca71488f5bbf498d92e4d1194f6f941307ffd95f7<br>5e76869f0e |

The following annotated hexdump shows a `spork` message.

~~~
11270000 .................................... Spork ID: Spork 2 InstantSend enabled (10001)
0000000000000000 ............................ Value (0)
2478da5900000000 ............................ Epoch time: 2017-10-08 19:10:28 UTC (1507489828)

41 .......................................... Signature length: 65

1b6762d3e70890b5cfaed5d1fd72121c
d32020c827a89f8128a00acd210f4ea4
1b36c26c3767f8a24f48663e189865ed
403ed1e850cdb4207cdd466419d9d183
45 .......................................... Masternode Signature
~~~


# VerAck

The `verack` message acknowledges a previously-received `version` message, informing the connecting node that it can begin to send other messages. The `verack` message has no payload; for an example of a message with no payload, see the [message headers section][section message header].


# Version

The `version` message provides information about the transmitting node to the receiving node at the beginning of a connection. Until both peers have exchanged `version` messages, no other messages will be accepted.

If a `version` message is accepted, the receiving node should send a `verack` message---but no node should send a `verack` message before initializing its half of the connection by first sending a `version` message.

Protocol version 70214 added a masternode authentication (challenge/response) system. Following the `verack` message, masternodes should send a `mnauth` message that signs the `mnauth_challenge` with their BLS operator key.


| Bytes    | Name                  | Data Type        | Required/Optional                        | Description
|----------|-----------------------|------------------|------------------------------------------|-------------
| 4        | version               | int32_t          | Required                                 | The highest protocol version understood by the transmitting node.  See the [protocol version section][section protocol versions].
| 8        | services              | uint64_t         | Required                                 | The services supported by the transmitting node encoded as a bitfield.  See the list of service codes below.
| 8        | timestamp             | int64_t          | Required                                 | The current Unix epoch time according to the transmitting node's clock.  Because nodes will reject blocks with timestamps more than two hours in the future, this field can help other nodes to determine that their clock is wrong.
| 8        | addr_recv services    | uint64_t         | Required                                 | *Added in protocol version 106.* <br><br>The services supported by the receiving node as perceived by the transmitting node.  Same format as the 'services' field above. Dash Core will attempt to provide accurate information.
| 16       | addr_recv IP address  | char             | Required                                 | *Added in protocol version 106.* <br><br>The IPv6 address of the receiving node as perceived by the transmitting node in **big endian byte order**. IPv4 addresses can be provided as [IPv4-mapped IPv6 addresses][]. Dash Core will attempt to provide accurate information.
| 2        | addr_recv port        | uint16_t         | Required                                 | *Added in protocol version 106.* <br><br>The port number of the receiving node as perceived by the transmitting node in **big endian byte order**.
| 8        | addr_trans services   | uint64_t         | Required                                 | The services supported by the transmitting node.  Should be identical to the 'services' field above.
| 16       | addr_trans IP address | char             | Required                                 | The IPv6 address of the transmitting node in **big endian byte order**. IPv4 addresses can be provided as [IPv4-mapped IPv6 addresses][].  Set to ::ffff:127.0.0.1 if unknown.
| 2        | addr_trans port       | uint16_t         | Required                                 | The port number of the transmitting node in **big endian byte order**.
| 8        | nonce                 | uint64_t         | Required                                 | A random nonce which can help a node detect a connection to itself.  If the nonce is 0, the nonce field is ignored.  If the nonce is anything else, a node should terminate the connection on receipt of a `version` message with a nonce it previously sent.
| *Varies* | user_agent bytes      | compactSize uint | Required                                 | Number of bytes in following user\_agent field.  If 0x00, no user agent field is sent.
| *Varies* | user_agent            | string           | Required if user_agent bytes > 0         | *Renamed in protocol version 60000.* <br><br>User agent as defined by BIP14. Previously called subVer.<br><br>Dash Core limits the length to 256 characters.
| 4        | start_height          | int32_t          | Required                                 | The height of the transmitting node's best block chain or, in the case of an SPV client, best block header chain.
| 1        | relay                 | bool             | Optional                                 | *Added in protocol version 70001 as described by BIP37.* <br><br>Transaction relay flag.  If 0x00, no `inv` messages or `tx` messages announcing new transactions should be sent to this client until it sends a `filterload` message or `filterclear` message.  If the relay field is not present or is set to 0x01, this node wants `inv` messages and `tx` messages announcing new transactions.
| 32       | mnauth_challenge      | uint256          | Optional                                 | *Added in protocol version 70214* <br><br>A challenge to be signed by the receiving masternode. The response is returned via a `mnauth` message following the `verack` message.


The following service identifiers have been assigned.

| Value | Name         | Description
|-------|--------------|---------------
| 0x00  | *Unnamed*    | This node is not a full node.  It may not be able to provide any data except for the transactions it originates.
| 0x01  | NODE_NETWORK | This is a full node and can be asked for full blocks.  It should implement all protocol features available in its self-reported protocol version.
| 0x02  | NODE_GETUTXO | This node is capable of responding to the getutxo protocol request. *Dash Core does not support this service.*
| 0x04  | NODE_BLOOM | This node is capable and willing to handle bloom-filtered connections.  Dash Core nodes used to support this by default, without advertising this bit, but no longer do as of protocol version 70201 (= NO_BLOOM_VERSION)

The following annotated hexdump shows a `version` message. (The message header has been omitted and the actual IP addresses have been replaced with [RFC5737][] reserved IP addresses.)

~~~
46120100 .................................... Protocol version: 70214
0500000000000000 ............................ Services: NODE_NETWORK (1) + NODE_BLOOM (4)
9c10ad5c00000000 ............................ Epoch time: 1554845852

0100000000000000 ............................ Receiving node's services
00000000000000000000ffffc61b6409 ............ Receiving node's IPv6 address
270f ........................................ Receiving node's port number

0500000000000000 ............................ Transmitting node's services
00000000000000000000ffffcb0071c0 ............ Transmitting node's IPv6 address
270f ........................................ Transmitting node's port number

128035cbc97953f8 ............................ Nonce

12 .......................................... Bytes in user agent string: 18
2f4461736820436f72653a302e31322e312e352f..... User agent: /Dash Core:0.14.0/

851f0b00 .................................... Start height: 76944
01 .......................................... Relay flag: true

5dbb5d1baade6a9afa34db708f72c0dd
b5bd82b3656493484556689640a91357 ............ Masternode Auth. Challenge
~~~