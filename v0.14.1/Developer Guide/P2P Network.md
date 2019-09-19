---
title: "P2P Network"
excerpt: ""
---
The Dash network protocol allows full nodes (peers) to collaboratively maintain a [peer-to-peer network][network]{:#term-network}{:.term} for block and transaction exchange. Full nodes download and verify every block and transaction prior to relaying them to other nodes. Archival nodes are full nodes which store the entire blockchain and can serve historical blocks to other nodes. Pruned nodes are full nodes which do not store the entire blockchain. Many SPV clients also use the Dash network protocol to connect to full nodes.

Consensus rules do _not_ cover networking, so Dash programs may use alternative networks and protocols, such as the [high-speed block relay network][] used by some miners and the [dedicated transaction information servers][electrum server] used by some wallets that provide SPV-level security.

To provide practical examples of the Dash peer-to-peer network, this section uses Dash Core as a representative full node and [DashJ][] as a representative SPV client. Both programs are flexible, so only default behavior is described. Also, for privacy, actual IP addresses in the example output below have been replaced with [RFC5737][] reserved IP addresses.