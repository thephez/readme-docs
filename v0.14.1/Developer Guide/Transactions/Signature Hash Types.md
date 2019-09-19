---
title: "Signature Hash Types"
excerpt: ""
---
`OP_CHECKSIG` extracts a non-stack argument from each signature it evaluates, allowing the signer to decide which parts of the transaction to sign. Since the signature protects those parts of the transaction from modification, this lets signers selectively choose to let other people modify their transactions.

The various options for what to sign are called [signature hash][/en/glossary/signature-hash]{:#term-signature-hash}{:.term} types. There are three base SIGHASH types currently available:

* [`SIGHASH_ALL`][/en/glossary/sighash-all]{:#term-sighash-all}{:.term}, the default, signs all the inputs and outputs, protecting everything except the signature scripts against modification.

* [`SIGHASH_NONE`][/en/glossary/sighash-none]{:#term-sighash-none}{:.term} signs all of the inputs but none of the outputs, allowing anyone to change where the duffs are going unless other signatures using other signature hash flags protect the outputs.

* [`SIGHASH_SINGLE`][/en/glossary/sighash-single]{:#term-sighash-single}{:.term} the only output signed is the one corresponding to this input (the output with the same output index number as this input), ensuring nobody can change your part of the transaction but allowing other signers to change their part of the transaction. The corresponding output must exist or the value "1" will be signed, breaking the security scheme. This input, as well as other inputs, are included in the signature. The sequence numbers of other inputs are not included in the signature, and can be updated.

The base types can be modified with the [`SIGHASH_ANYONECANPAY`][/en/glossary/sighash-anyonecanpay]{:#term-sighash-anyonecanpay}{:.term} (anyone can pay) flag, creating three new combined types:

* `SIGHASH_ALL|SIGHASH_ANYONECANPAY` signs all of the outputs but only this one input, and it also allows anyone to add or remove other inputs, so anyone can contribute additional duffs but they cannot change how many duffs are sent nor where they go.

* `SIGHASH_NONE|SIGHASH_ANYONECANPAY` signs only this one input and allows anyone to add or remove other inputs or outputs, so anyone who gets a copy of this input can spend it however they'd like.

* `SIGHASH_SINGLE|SIGHASH_ANYONECANPAY` signs this one input and its corresponding output. Allows anyone to add or remove other inputs.

Because each input is signed, a transaction with multiple inputs can have multiple signature hash types signing different parts of the transaction. For example, a single-input transaction signed with `NONE` could have its output changed by the miner who adds it to the block chain. On the other hand, if a two-input transaction has one input signed with `NONE` and one input signed with `ALL`, the `ALL` signer can choose where to spend the duffs without consulting the `NONE` signer---but nobody else can modify the transaction.