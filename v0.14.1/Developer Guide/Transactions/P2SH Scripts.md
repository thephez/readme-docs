---
title: "P2SH Scripts"
excerpt: ""
---
Pubkey scripts are created by spenders who have little interest what that script does. Receivers do care about the script conditions and, if they want, they can ask spenders to use a particular pubkey script. Unfortunately, custom pubkey scripts are less convenient than short Dash addresses and there was no standard way to communicate them between programs prior to widespread implementation of the BIP70 Payment Protocol discussed later.

To solve these problems, pay-to-script-hash ([P2SH][/en/glossary/p2sh-address]{:#term-p2sh}{:.term}) transactions were created in 2012 to let a spender create a pubkey script containing a hash of a second script, the [redeem script][/en/glossary/redeem-script]{:#term-redeem-script}{:.term}.

The basic P2SH workflow, illustrated below, looks almost identical to the P2PKH workflow. Bob creates a redeem script with whatever script he wants, hashes the redeem script, and provides the redeem script hash to Alice. Alice creates a P2SH-style output containing Bob's redeem script hash.

![Creating A P2SH Redeem Script And Hash](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-creating-p2sh-output.png)

When Bob wants to spend the output, he provides his signature along with the full (serialized) redeem script in the signature script. The peer-to-peer network ensures the full redeem script hashes to the same value as the script hash Alice put in her output; it then processes the redeem script exactly as it would if it were the primary pubkey script, letting Bob spend the output if the redeem script does not return false.

![Unlocking A P2SH Output For Spending](https://github.com/dash-docs/dash-docs/raw/master/img/dev/en-unlocking-p2sh-output.png)

The hash of the redeem script has the same properties as a pubkey hash---so it can be transformed into the standard Dash address format with only one small change to differentiate it from a standard address. This makes collecting a P2SH-style address as simple as collecting a P2PKH-style address. The hash also obfuscates any public keys in the redeem script, so P2SH scripts are as secure as P2PKH pubkey hashes.