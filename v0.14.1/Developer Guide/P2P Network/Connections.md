---
title: "Connections"
excerpt: ""
---
# Peer Discovery

When started for the first time, programs don't know the IP addresses of any active full nodes. In order to discover some IP addresses, they query one or more DNS names (called [DNS seeds][/en/glossary/dns-seed]{:#term-dns-seed}{:.term}) hardcoded into Dash Core. The response to the lookup should include one or more [DNS A records][] with the IP addresses of full nodes that may accept new incoming connections. For example, using the [Unix `dig`command][dig command]:

    ;; QUESTION SECTION:
    ;dnsseed.masternode<!--noref-->.io.		  IN	A

    ;; ANSWER SECTION:
    dnsseed.masternode<!--noref-->.io.	60	IN	A	192.0.2.113
    dnsseed.masternode<!--noref-->.io.	60	IN	A	198.51.100.231
    dnsseed.masternode<!--noref-->.io.	60	IN	A	203.0.113.183

    [...]

The DNS seeds are maintained by Dash community members: some of them provide dynamic DNS seed servers which automatically get IP addresses of active nodes by scanning the network; others provide static DNS seeds that are updated manually and are more likely to provide IP addresses for inactive nodes. In either case, nodes are added to the DNS seed if they run on the default Dash ports of 9999 for mainnet or 19999 for testnet.

DNS seed results are not authenticated and a malicious seed operator or network man-in-the-middle attacker could return only IP addresses of nodes controlled by the attacker, isolating a program on the attacker's own network and allowing the attacker to feed it bogus transactions and blocks. For this reason, programs should not rely on DNS seeds exclusively.

Once a program has connected to the network, its peers can begin to send it `addr` (address) messages with the IP addresses and port numbers of other peers on the network, providing a fully decentralized method of peer discovery. Dash Core keeps a record of known peers in a persistent on-disk database which usually allows it to connect directly to those peers on subsequent startups without having to use DNS seeds.

However, peers often leave the network or change IP addresses, so programs may need to make several different connection attempts at startup before a successful connection is made. This can add a significant delay to the amount of time it takes to connect to the network, forcing a user to wait before sending a transaction or checking the status of payment.

Dash Core tries to strike a balance between minimizing delays and avoiding unnecessary DNS seed use: if Dash Core has entries in its peer database, it spends up to 11 seconds attempting to connect to at least one of them before falling back to seeds; if a connection is made within that time, it does not query any seeds.

Dash Core also includes a hardcoded list of IP addresses and port numbers to several dozen nodes which were active around the time that particular version of the software was first released. Starting with Dash Core 0.12.3, masternodes are used for the seed list since they must remain online to receive their portion of the block reward (good availability) and must be compliant with consensus rules (reliable). Dash Core will start attempting to connect to these nodes if none of the DNS seed servers have responded to a query within 60 seconds, providing an automatic fallback option.

As a manual fallback option, Dash Core also provides several command-line connection options, including the ability to get a list of peers from a specific node by IP address, or to make a persistent connection to a specific node by IP address.  See the `-help` text for details.

**Resources:** [Dash Seeder][], the program run by several of the seeds used by Dash Core. The Dash Core [DNS Seed Policy][]. The hardcoded list  of IP addresses used by Dash Core is generated using the [makeseeds script][].

# Connecting To Peers

Connecting to a peer is done by sending a `version` message, which contains your version number, block, and current time to the remote node. The remote node responds with its own `version` message. Then both nodes send a `verack` message to the other node to indicate the connection has been established.

Once connected, the client can send to the remote node `getaddr` and `addr` messages to gather additional peers.

In order to maintain a connection with a peer, nodes by default will send a message to peers before 30 minutes of inactivity. If 90 minutes pass without a message being received by a peer, the client will assume that connection has closed.