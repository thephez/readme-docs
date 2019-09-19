---
title: "Blocks"
excerpt: ""
---
# Initial Block Download
 
 Before a full node can validate unconfirmed transactions and recently-mined blocks, it must download and validate all blocks from block 1 (the block after the hardcoded genesis block) to the current tip of the best block chain. This is the Initial Block Download (IBD) or initial sync.

Although the word "initial" implies this method is only used once, it can also be used any time a large number of blocks need to be downloaded, such as when a previously-caught-up node has been offline for a long time. In this case, a node can use the IBD method to download all the blocks which were produced since the last time it was online.

Dash Core uses the IBD method any time the last block on its local best block chain has a block header time more than 24 hours in the past. Dash Core will also perform IBD if its local best block chain is more than 144 blocks lower than its local best header chain (that is, the local block chain is more than about 6 hours in the past).

## Blocks-First
 
 Dash Core (up until version 0.12.0.x) uses a simple initial block download (IBD) method we'll call *blocks-first*. The goal is to download the blocks from the best block chain in sequence.

![Overview Of Blocks-First Method](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-blocks-first-flowchart.png)

The first time a node is started, it only has a single block in its local best block chain---the hardcoded genesis block (block 0).  This node chooses a remote peer, called the sync node, and sends it the `getblocks` message illustrated below.

![First GetBlocks Message Sent During IBD](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-getblocks.png)

In the header hashes field of the `getblocks` message, this new node sends the header hash of the only block it has, the genesis block (b67a...0000 in internal byte order).  It also sets the stop hash field to all zeroes to request a maximum-size response.

Upon receipt of the `getblocks` message, the sync node takes the first (and only) header hash and searches its local best block chain for a block with that header hash. It finds that block 0 matches, so it replies with 500 block inventories (the maximum response to a `getblocks` message) starting from block 1. It sends these inventories in the `inv` message illustrated below.

![First Inv Message Sent During IBD](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-inv.png)

Inventories are unique identifiers for information on the network. Each inventory contains a type field and the unique identifier for an instance of the object. For blocks, the unique identifier is a hash of the block's header.

The block inventories appear in the `inv` message in the same order they appear in the block chain, so this first `inv` message contains inventories for blocks 1 through 501. (For example, the hash of block 1 is 4343...0000 as seen in the illustration above.)

The IBD node uses the received inventories to request 128 blocks from the sync node in the `getdata` message illustrated below.

![First GetData Message Sent During IBD](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-getdata.png)

It's important to blocks-first nodes that the blocks be requested and sent in order because each block header references the header hash of the preceding block. That means the IBD node can't fully validate a block until its parent block has been received. Blocks that can't be validated because their parents haven't been received are called orphan blocks; a subsection below describes them in more detail.

Upon receipt of the `getdata` message, the sync node replies with each of the blocks requested. Each block is put into serialized block format and sent in a separate `block` message. The first `block` message sent (for block 1) is illustrated below.

![First Block Message Sent During IBD](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-block.png)

The IBD node downloads each block, validates it, and then requests the next block it hasn't requested yet, maintaining a queue of up to 128 blocks to download. When it has requested every block for which it has an inventory, it sends another `getblocks` message to the sync node requesting the inventories of up to 500 more blocks.  This second `getblocks` message contains multiple header hashes as illustrated below:

![Second GetBlocks Message Sent During IBD](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-getblocks2.png)

Upon receipt of the second `getblocks` message, the sync node searches its local best block chain for a block that matches one of the header hashes in the message, trying each hash in the order they were received. If it finds a matching hash, it replies with 500 block inventories starting with the next block from that point. But if there is no matching hash (besides the stopping hash), it assumes the only block the two nodes have in common is block 0 and so it sends an `inv` starting with block 1 (the same `inv` message seen several illustrations above).

This repeated search allows the sync node to send useful inventories even if the IBD node's local block chain forked from the sync node's local block chain. This fork detection becomes increasingly useful the closer the IBD node gets to the tip of the block chain.

When the IBD node receives the second `inv` message, it will request those blocks using `getdata` messages.  The sync node will respond with `block` messages.  Then the IBD node will request more inventories with another `getblocks` message---and the cycle will repeat until the IBD node is synced to the tip of the block chain.  At that point, the node will accept blocks sent through the regular block broadcasting described in a later subsection.

### Blocks-First Advantages & Disadvantages
 
 The primary advantage of blocks-first IBD is its simplicity. The primary disadvantage is that the IBD node relies on a single sync node for all of its downloading. This has several implications:

* **Speed Limits:** All requests are made to the sync node, so if the sync node has limited upload bandwidth, the IBD node will have slow download speeds.  Note: if the sync node goes offline, Dash Core will continue downloading from another node---but it will still only download from a single sync node at a time.

* **Download Restarts:** The sync node can send a non-best (but otherwise valid) block chain to the IBD node. The IBD node won't be able to identify it as non-best until the initial block download nears completion, forcing the IBD node to restart its block chain download over again from a different node. Dash Core ships with several block chain checkpoints at various block heights selected by developers to help an IBD node detect that it is being fed an alternative block chain history---allowing the IBD node to restart its download earlier in the process.

* **Disk Fill Attacks:** Closely related to the download restarts, if the sync node sends a non-best (but otherwise valid) block chain, the chain will be stored on disk, wasting space and possibly filling up the disk drive with useless data.

* **High Memory Use:** Whether maliciously or by accident, the sync node can send blocks out of order, creating orphan blocks which can't be validated until their parents have been received and validated. Orphan blocks are stored in memory while they await validation, which may lead to high memory use.

All of these problems are addressed in part or in full by the headers-first IBD method used in Dash Core 0.12.0.x.

**Resources:** The table below summarizes the messages mentioned throughout this subsection. The links in the message field will take you to the reference page for that message.

| **Message** | `getblocks` message | `inv` message                             | `getdata` message  | `block` message
| - | - | - |
| **From→To** | IBD→Sync                         | Sync→IBD                                         | IBD→Sync                      | Sync→IBD
| **Payload** | One or more header hashes        | Up to 500 block inventories (unique identifiers) | One or more block inventories | One serialized block

## Headers-First
 
 Dash Core 0.12.0 uses an initial block download (IBD) method called *headers-first*. The goal is to download the headers for the best [header chain][/en/glossary/header-chain]{:#term-header-chain}{:.term}, partially validate them as best as possible, and then download the corresponding blocks in parallel.  This solves several problems with the older blocks-first IBD method.

![Overview Of Headers-First Method](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-headers-first-flowchart.png)

The first time a node is started, it only has a single block in its local best block chain---the hardcoded genesis block (block 0).  The node chooses a remote peer, which we'll call the sync node, and sends it the `getheaders` message illustrated below.

![First getheaders message](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-getheaders.png)

In the header hashes field of the `getheaders` message, the new node sends the header hash of the only block it has, the genesis block (b67a...0000 in internal byte order).  It also sets the stop hash field to all zeroes to request a maximum-size response.

Upon receipt of the `getheaders` message, the sync node takes the first (and only) header hash and searches its local best block chain for a block with that header hash. It finds that block 0 matches, so it replies with 2,000 header (the maximum response) starting from block 1. It sends these header hashes in the `headers` message illustrated below.

![First headers message](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-ibd-headers.png)

The IBD node can partially validate these block headers by ensuring that all fields follow consensus rules and that the hash of the header is below the target threshold according to the nBits field.  (Full validation still requires all transactions from the corresponding block.)

After the IBD node has partially validated the block headers, it can do two things in parallel:

1. **Download More Headers:** the IBD node can send another `getheaders` message to the sync node to request the next 2,000 headers on the best header chain. Those headers can be immediately validated and another batch requested repeatedly until a `headers` message is received from the sync node with fewer than 2,000 headers, indicating that it has no more headers to offer. As of this writing, headers sync can be completed in fewer than 200 round trips, or about 32 MB of downloaded data.

    Once the IBD node receives a `headers` message with fewer than 2,000 headers from the sync node, it sends a `getheaders` message to each of its outbound peers to get their view of best header chain. By comparing the responses, it can easily determine if the headers it has downloaded belong to the best header chain reported by any of its outbound peers. This means a dishonest sync node will quickly be discovered even if checkpoints aren't used (as long as the IBD node connects to at least one honest peer; Dash Core will continue to provide checkpoints in case honest peers can't be found).

2. **Download Blocks:** While the IBD node continues downloading headers, and after the headers finish downloading, the IBD node will request and download each block. The IBD node can use the block header hashes it computed from the header chain to create `getdata` messages that request the blocks it needs by their inventory. It doesn't need to request these from the sync node---it can request them from any of its full node peers. (Although not all full nodes may store all blocks.) This allows it to fetch blocks in parallel and avoid having its download speed constrained to the upload speed of a single sync node.

    To spread the load between multiple peers, Dash Core will only request up to 16 blocks at a time from a single peer. Combined with its maximum of 8 outbound connections, this means headers-first Dash Core will request a maximum of 128 blocks simultaneously during IBD (the same maximum number that blocks-first Dash Core requested from its sync node).

![Simulated Headers-First Download Window](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-headers-first-moving-window.png)

Dash Core's headers-first mode uses a 1,024-block moving download window to maximize download speed. The lowest-height block in the window is the next block to be validated; if the block hasn't arrived by the time Dash Core is ready to validate it, Dash Core will wait a minimum of two more seconds for the stalling node to send the block. If the block still hasn't arrived, Dash Core will disconnect from the stalling node and attempt to connect to another node. For example, in the illustration above, Node A will be disconnected if it doesn't send block 3 within at least two seconds.

Once the IBD node is synced to the tip of the block chain, it will accept blocks sent through the regular block broadcasting described in a later subsection.

**Resources:** The table below summarizes the messages mentioned throughout this subsection. The links in the message field will take you to the reference page for that message.

| **Message** | [`getheaders`][getheaders message] | [`headers`][headers message] | [`getdata`][getdata message]                             | [`block`][block message]
| - | - | - |
| **From→To** | IBD→Sync                           | Sync→IBD                     | IBD→*Many*                                               | *Many*→IBD
| **Payload** | One or more header hashes          | Up to 2,000 block headers    | One or more block inventories derived from header hashes | One serialized block

# Block Broadcasting
 
 When a miner discovers a new block, it broadcasts the new block to its peers using one of the following methods:

* **[Unsolicited Block Push][]{:#term-unsolicited-block-push}{:.term}:** the miner sends a `block` message to each of its full node peers with the new block. The miner can reasonably bypass the standard relay method in this way because it knows none of its peers already have the just-discovered block.

* **[Standard Block Relay][]{:#term-standard-block-relay}{:.term}:** the miner, acting as a standard relay node, sends an `inv` message to each of its peers (both full node and SPV) with an inventory referring to the new block. The most common responses are:

   * Each blocks-first (BF) peer that wants the block replies with a `getdata` message requesting the full block.

   * Each headers-first (HF) peer that wants the block replies with a `getheaders` message containing the header hash of the highest-height header on its best header chain, and likely also some headers further back on the best header chain to allow fork detection. That message is immediately followed by a `getdata` message requesting the full block. By requesting headers first, a headers-first peer can refuse orphan blocks as described in the subsection below.

   * Each Simplified Payment Verification (SPV) client that wants the block replies with a `getdata` message typically requesting a merkle block. 
   
   The miner replies to each request accordingly by sending the block in a `block` message, one or more headers in a `headers` message, or the merkle block and transactions relative to the SPV client's bloom filter in a `merkleblock` message followed by zero or more `tx` messages.

By default, Dash Core broadcasts blocks using standard block relay,but it will accept blocks sent using either of the methods described above.

Full nodes validate the received block and then advertise it to theirpeers using the standard block relay method described above.  The condensed table below highlights the operation of the messages described above(Relay, BF, HF, and SPV refer to the relay node, a blocks-first node, a headers-first node, and an SPV client; *any* refers to a node using any block retrieval method.)

| **Message** | `inv` message                                   | `getdata` message               | `getheaders` message                                     | `headers` message
| - | - | - |
| **From→To** | Relay→_Any_                                            | BF→Relay                                   | HF→Relay                                                               | Relay→HF
| **Payload** | The inventory of the new block                         | The inventory of the new block             | One or more header hashes on the HF node's best header chain (BHC)     | Up to 2,000 headers connecting HF node's BHC to relay node's BHC
| **Message** | [`block`][block message]                               | [`merkleblock`][merkleblock message]       | [`tx`][tx message]                                                     |
| **From→To** | Relay→BF/HF                                            | Relay→SPV                                  | Relay→SPV                                                              |
| **Payload** | The new block in [serialized format][section serialized blocks] | The new block filtered into a merkle block | Serialized transactions from the new block that match the bloom filter |

## Orphan Blocks
 
 Blocks-first nodes may download orphan blocks---blocks whose previous block header hash field refers to a block header this node hasn't seen yet. In other words, orphan blocks have no known parent(unlike stale blocks, which have known parents but which aren't part of the best block chain).

![Difference Between Orphan And Stale Blocks](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-orphan-stale-definition.png)

When a blocks-first node downloads an orphan block, it will not validate it. Instead, it will send a `getblocks` message to the node which sent the orphan block; the broadcasting node will respond with an `inv` message containing inventories of any blocks the downloading node is missing (upto 500); the downloading node will request those blocks with a `getdata`message; and the broadcasting node will send those blocks with a `block`message. The downloading node will validate those blocks, and once theparent of the former orphan block has been validated, it will validate the former orphan block.

Headers-first nodes avoid some of this complexity by always requesting block headers with the `getheaders` message before requesting a block with the `getdata` message. The broadcasting node will send a `headers`message containing all the block headers (up to 2,000) it thinks the downloading node needs to reach the tip of the best header chain; each of those headers will point to its parent, so when the downloading node receives the `block` message, the block shouldn't be an orphan block---all of its parents should be known (even if they haven't been validated yet). If, despite this, the block received in the `block`message is an orphan block, a headers-first node will discard it immediately.

However, orphan discarding does mean that headers-first nodes will ignore orphan blocks sent by miners in an unsolicited block push.