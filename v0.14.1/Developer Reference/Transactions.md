---
title: "Transactions"
excerpt: ""
---
# Opcodes

The opcodes used in the pubkey scripts of standard transactions are:

* Various data pushing opcodes from 0x00 to 0x4e (1--78). These aren't typically shown in examples, but they must be used to push signatures and public keys onto the stack. See the link below this list for a description.

* `OP_TRUE`/`OP_1` (0x51) and `OP_2` through `OP_16` (0x52--0x60), which push the values 1 through 16 to the stack.

* [`OP_CHECKSIG`][op_checksig]{:#term-op-checksig}{:.term} (0xac) consumes a signature and a full public key, and pushes true onto the stack if the transaction data specified by the SIGHASH flag was converted into the signature using the same ECDSA private key that generated the public key.  Otherwise, it pushes false onto the stack.

* [`OP_DUP`][op_dup]{:#term-op-dup}{:.term} (0x76) pushes a copy of the topmost stack item on to the stack.

* [`OP_HASH160`][op_hash160]{:#term-op-hash160}{:.term} (0xa9) consumes the topmost item on the stack, computes the RIPEMD160(SHA256()) hash of that item, and pushes that hash onto the stack.

* [`OP_EQUAL`][op_equal]{:#term-op-equal}{:.term} (0x87) consumes the top two items on the stack, compares them, and pushes true onto the stack if they are the same, false if not.

* [`OP_VERIFY`][op_verify]{:#term-op-verify}{:.term} (0x69) consumes the topmost item on the stack.
  If that item is zero (false) it terminates the script in failure.

* [`OP_EQUALVERIFY`][op_equalverify]{:#term-op-equalverify}{:.term} (0x88) runs `OP_EQUAL` and then `OP_VERIFY` in sequence.

* [`OP_CHECKMULTISIG`][op_checkmultisig]{:#term-op-checkmultisig}{:.term} (0xae) consumes the value (n) at the top of the stack, consumes that many of the next stack levels (public keys), consumes the value (m) now at the top of the stack, and consumes that many of the next values (signatures) plus one extra value.

    The "one extra value" it consumes is the result of an off-by-one error in the Bitcoin Core implementation. This value is not used, so signature scripts prefix the list of secp256k1 signatures with a single OP_0 (0x00).

    `OP_CHECKMULTISIG` compares the first signature against each public key until it finds an ECDSA match. Starting with the subsequent public key, it compares the second signature against each remaining public key until it finds an ECDSA match. The process is repeated until all signatures have been checked or not enough public keys remain to produce a successful result.

    Because public keys are not checked again if they fail any signature comparison, signatures must be placed in the signature script using the same order as their corresponding public keys were placed in the pubkey script or redeem script. See the `OP_CHECKMULTISIG` warning below for more details.

* [`OP_RETURN`][op_return]{:#term-op-return}{:.term} (0x6a) terminates the script in failure when executed.

A complete list of opcodes can be found on the Bitcoin Wiki [Script Page][wiki script], with an authoritative list in the `opcodetype` enum of the Dash Core [script header file][core script.h]

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) **<span id="signature_script_modification_warning">Signature script modification warning</span>:**
Signature scripts are not signed, so anyone can modify them. This means signature scripts should only contain data and data-pushing opcodes which can't be modified without causing the pubkey script to fail.
Placing non-data-pushing opcodes in the signature script currently makes a transaction non-standard, and future consensus rules may forbid such transactions altogether. (Non-data-pushing opcodes are already forbidden in signature scripts when spending a P2SH pubkey script.)

![Warning icon](https://github.com/dash-docs/dash-docs/raw/master/img/icons/icon_warning.png) **`OP_CHECKMULTISIG` warning:** The multisig verification process described above requires that signatures in the signature script be provided in the same order as their corresponding public keys in the pubkey script or redeem script. For example, the following combined signature and pubkey script will produce the stack and comparisons shown:

~~~
OP_0 <A sig> <B sig> OP_2 <A pubkey> <B pubkey> <C pubkey> OP_3

Sig Stack       Pubkey Stack  (Actually a single stack)
---------       ------------
B sig           C pubkey
A sig           B pubkey
OP_0            A pubkey

1. B sig compared to C pubkey (no match)
2. B sig compared to B pubkey (match #1)
3. A sig compared to A pubkey (match #2)

Success: two matches found
~~~

But reversing the order of the signatures with everything else the same
will fail, as shown below:

~~~
OP_0 <B sig> <A sig> OP_2 <A pubkey> <B pubkey> <C pubkey> OP_3

Sig Stack       Pubkey Stack  (Actually a single stack)
---------       ------------
A sig           C pubkey
B sig           B pubkey
OP_0            A pubkey

1. A sig compared to C pubkey (no match)
2. A sig compared to B pubkey (no match)

Failure, aborted: two signature matches required but none found so
                  far, and there's only one pubkey remaining
~~~

# Address Conversion

The hashes used in P2PKH and P2SH outputs are commonly encoded as Dash addresses.  This is the procedure to encode those hashes and decode the addresses.

First, get your hash.  For P2PKH, you RIPEMD-160(SHA256()) hash a ECDSA public key derived from your 256-bit ECDSA private key (random data). For P2SH, you RIPEMD-160(SHA256()) hash a redeem script serialized in the format used in raw transactions (described in a [following sub-section][raw transaction format]).  Taking the resulting hash:

1. Add an address version byte in front of the hash.  The version bytes commonly used by Dash are:

    * 0x4c for P2PKH addresses on the main Dash network (mainnet)

    * 0x8c for P2PKH addresses on the Dash testing network (testnet)

    * 0x10 for P2SH addresses on mainnet

    * 0x13 for P2SH addresses on testnet

2. Create a copy of the version and hash; then hash that twice with SHA256: `SHA256(SHA256(version . hash))`

3. Extract the first four bytes from the double-hashed copy. These are used as a checksum to ensure the base hash gets transmitted correctly.

4. Append the checksum to the version and hash, and encode it as a base58 string: `BASE58(version . hash . checksum)`

Dash's base58 encoding, called [Base58Check][/en/glossary/base58check]{:#term-base58check}{:.term} may not match other implementations. Tier Nolan provided the following example encoding algorithm to the Bitcoin Wiki [Base58Check encoding](https://en.bitcoin.it/wiki/Base58Check_encoding) page under the [Creative Commons Attribution 3.0 license](https://creativecommons.org/licenses/by/3.0/):

```c
code_string = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz"
x = convert_bytes_to_big_integer(hash_result)

output_string = ""

while(x > 0)
   {
       (x, remainder) = divide(x, 58)
       output_string.append(code_string[remainder])
   }

repeat(number_of_leading_zero_bytes_in_hash)
   {
   output_string.append(code_string[0]);
   }

output_string.reverse();
```

Dash's own code can be traced using the [base58 header file][core base58.h].

To convert addresses back into hashes, reverse the base58 encoding, extract the checksum, repeat the steps to create the checksum and compare it against the extracted checksum, and then remove the version byte.

# CompactSize Unsigned Integers

The raw transaction format and several peer-to-peer network messages use
a type of variable-length integer to indicate the number of bytes in a
following piece of data.

Dash Core code and this document refers to these variable length integers as compactSize. Many other documents refer to them as var_int or varInt, but this risks conflation with other variable-length integer encodings---such as the CVarInt class used in Dash Core for serializing data to disk.  Because it's used in the transaction format, the format of compactSize unsigned integers is part of the consensus rules.

For numbers from 0 to 252 (0xfc), compactSize unsigned integers look like regular unsigned integers. For other numbers up to 0xffffffffffffffff, a byte is prefixed to the number to indicate its length---but otherwise the numbers look like regular unsigned integers in little-endian order.

| Value                                   | Bytes Used | Format
|-----------------------------------------|------------|-----------------------------------------
| >= 0 && <= 0xfc (252)                   | 1          | uint8_t
| >= 0xfd (253) && <= 0xffff              | 3          | 0xfd followed by the number as uint16_t
| >= 0x10000 && <= 0xffffffff             | 5          | 0xfe followed by the number as uint32_t
| >= 0x100000000 && <= 0xffffffffffffffff | 9          | 0xff followed by the number as uint64_t

For example, the number 515 is encoded as 0xfd0302.