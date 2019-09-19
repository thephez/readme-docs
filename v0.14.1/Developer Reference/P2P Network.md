---
title: "P2P Network"
excerpt: "Dash P2P protocol"
---
This section describes the Dash P2P network protocol (but it is not a specification). It does not describe the [BIP70 payment protocol][/en/glossary/payment-protocol], the [GetBlockTemplate mining protocol][section getblocktemplate], or any network protocol never implemented in an official version of Dash Core.

All peer-to-peer communication occurs entirely over TCP.

~~~
Note: unless their description says otherwise, all multi-byte
integers mentioned in this section are transmitted in little-endian order.
~~~

# Constants and Defaults

The following constants and defaults are taken from Dash Core's
[chainparams.cpp][core chainparams.cpp] source code file.

| Network | Default Port | Magic Value | Start String | Max nBits
|---------|--------------|-----------------------------------------------|---------------
| Mainnet | 9999         | 0xBD6B0CBF  | 0xBF0C6BBD                      | 0x1e0ffff0
| Testnet | 19999        | 0xFFCAE2CE  | 0xCEE2CAFF                      | 0x1e0ffff0
| Regtest | 19994        | 0xDCB7C1FC  | 0xFCC1B7DC                      | 0x207fffff
| Devnet  | User-defined | 0xCEFFCAE2  | 0xE2CAFFCE                      | 0x207fffff

Note: the testnet start string and nBits above are for testnet3.

Command line parameters can change what port a node listens on (see `-help`). Start strings are hardcoded constants that appear at the start of all messages sent on the Dash network; they may also appear in data files such as Dash Core's block database. The Magic Value and nBits displayed above are in big-endian order; they're sent over the network in little-endian order. The Start String is simply the endian reversed Magic Value.

Dash Core's [chainparams.cpp][core chainparams.cpp] also includes other constants useful to programs, such as the hash of the genesis blocks for the different networks.

# Protocol Versions

The table below lists some notable versions of the P2P network protocol, with the most recent versions listed first. (If you know of a protocol version that implemented a major change but which is not listed here, please [open an issue][docs issue].)

As of Dash Core 0.14.0, the most recent protocol version is 70215.

| Version | Initial Release                    | Major Changes
|---------|------------------------------------|--------------
| 70215  | Dash Core 0.14.0.1 <br>(May 2019)  | • None (Governance bugfix only)
| 70214  | Dash Core 0.14.0.0 <br>(May 2019)  | • Long-living Masternode Quorums<br>• ChainLocks<br>• PrivateSend improvements<br>• Experimental LLMQ InstantSend<br>• Bitcoin Core 0.15 backports
| 70213  | Dash Core 0.13.0.x <br>(Jan 2019)  | • Special Transactions<br>• Deterministic Masternode List<br>• Coinbase Special Transaction<br>• Automatic InstantSend
| 70210  | Dash Core 0.12.3.x <br>(July 2018)  | • Named Devnets<br>• New signature format / Spork 6 addition<br>• Bitcoin Core 0.13/0.14 backports<br>• [BIP90][]: Buried deployments<br>• [BIP147][]: NULLYDUMMY enforcement<br>• [BIP152][] Compact Blocks<br>• Transaction version increased to 2<br>• Zero fee transactions removed<br>• Pruning in Lite Mode
| 70208  | Dash Core 0.12.2.x <br>(Nov 2017)  | • [DIP1][] (2MB blocks)<br>• Fee reduction (10x)<br>• InstantSend fix<br>• PrivateSend improvements<br>• _Experimental_ HD wallet<br>• Local Masternode support removed
| 70206  | Dash Core 0.12.1.x <br>(Mar 2017)  | • Switch to Bitcoin Core 0.12.1<br>• BIP-0065 (CheckLockTimeVerify)<br>• BIP-0112 (CheckSequenceVerify)
| 70103  | Dash Core 0.12.0.x <br>(Aug 2015)  | • Switch to Bitcoin Core 0.10<br>• Decentralized budget system<br>• New IX implementation
| 70076  | Dash Core 0.11.2.x <br>(Mar 2015)  | • Masternode enhancements<br>• Mining/relay policy enhancements<br>• BIP-66 - strict DER encoding for signatures
| 70066  | Dash Core 0.11.1.x <br>(Feb 2015)  | • InstantX fully implemented<br>• Spork fully implemented<br>• Masternode payment updates<br>• Rebrand to Dash (0.11.1.26)
| 70052  | Dash Core 0.11.0.x <br>(Jan 2015)  | • Switch from fork of Litecoin 0.8 to Bitcoin 0.9.3<br>• Rebrand to Darkcoin Core
| 70051  | Dash Core 0.10.0.x <br>(Sep 2014)  | • Release of the originally closed source implementation of DarkSend
| 70002  | Dash Core 0.9.0.x <br>(Mar 2014)   | • Masternode implementation<br>• Rebrand to Darkcoin
| 70002  | Dash Core 0.8.7 <br>(Jan 2014)     | Initial release of Dash (branded XCoin) as a fork of Litecoin 0.8

Historical Bitcoin protocol versions for reference shown below since Dash is a
fork of Bitcoin Core.

| Version | Initial Release                    | Major Changes
|---------|------------------------------------|--------------
| 70012   | Bitcoin Core 0.12.0 <br>(Feb 2016) | [BIP130][]: <br>• Added `sendheaders` message.
| 70011   | Bitcoin Core 0.12.0 <br>(Feb 2016) | [BIP111][]: <br>• `filter*` messages are disabled without NODE_BLOOM after and including this version.
| 70002   | Bitcoin Core 0.9.0 <br>(Mar 2014)  | • Send multiple `inv` messages in response to a `mempool` message if necessary <br><br>[BIP61][]: <br>• Added `reject` message
| 70001   | Bitcoin Core 0.8.0 <br>(Feb 2013)  | • Added `notfound` message. <br><br>[BIP37][]: <br>• Added `filterload` message. <br>• Added `filteradd` message. <br>• Added `filterclear` message. <br>• Added `merkleblock` message. <br>• Added relay field to `version` message <br>• Added `MSG_FILTERED_BLOCK` inventory type to `getdata` message.
| 60002   | Bitcoin Core 0.7.0 <br>(Sep 2012)  | [BIP35][]: <br>• Added `mempool` message. <br>• Extended `getdata` message to allow download of memory pool transactions
| 60001   | Bitcoin Core 0.6.1 <br>(May 2012)  | [BIP31][]: <br>• Added nonce field to `ping` message <br>• Added `pong` message
| 60000   | Bitcoin Core 0.6.0 <br>(Mar 2012)  | [BIP14][]: <br>• Separated protocol version from Bitcoin Core version
| 31800   | Bitcoin Core 0.3.18 <br>(Dec 2010) | • Added `getheaders` message and `headers` message.
| 31402   | Bitcoin Core 0.3.15 <br>(Oct 2010) | • Added time field to `addr` message.
| 311     | Bitcoin Core 0.3.11 <br>(Aug 2010) | • Added `alert` message.
| 209     | Bitcoin Core 0.2.9 <br>(May 2010)  | • Added checksum field to message headers.
| 106     | Bitcoin Core 0.1.6 <br>(Oct 2009)  | • Added receive IP address fields to `version` message.