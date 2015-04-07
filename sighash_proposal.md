## SIGHASH Proposal

This document is the result of an IRC discussion from 4/7/15 with gmaxwell, tdryja, and StephenM347, on how to enable the bitcoin lightning network with more fully capable set of SIGHASH flags.

The current set if SIGHASH parameters are defined by the [below code snippet](https://github.com/bitcoin/bitcoin/blob/v0.10.0/src/script/interpreter.h#L21-L28):

```
/** Signature hash types/flags */
enum
{
    SIGHASH_ALL = 1,
    SIGHASH_NONE = 2,
    SIGHASH_SINGLE = 3,
    SIGHASH_ANYONECANPAY = 0x80,
};
```

However, these could be extended to provide greater flexebility. In particular, for the bitcoin lightning network, it would be useful to sign transactions without serializing in the TXID. In addition, it would be useful both for the lightning network and for hardware wallets to include the value of the UTXO being spent in the serialized and signed transaction, as was proposed in [this bitcointalk thread](https://bitcointalk.org/index.php?topic=181734.0). 

We propose using a 2-byte SIGHASH flag. One byte determines the inclusion/exclusion of data at the input for the executing `scriptSig` (and the output at the same index, if such an output exists). The second byte determines what data from other inputs/outputs should be serialized for the hash to sign. 

The new SIGHASH flags are described below.

```
/** Signature hash types/flags */
enum
{
    SIGHASH_WITHOUTTXID = 1,
    SIGHASH_WITHOUTINPUTVALUE = 2,
    SIGHASH_WITHOUTPREVSCRIPTPUBKEY = 4,
    SIGHASH_ANYONECANPAY = 0x80,
};
```

To soft-fork compatible, obviously these changes need to be done through an alternative method to using OP_CHECKSIG. Two such alternatives are:

 - OP_P3SH - Use a scriptPubKey similar to P2SH, but ending with OP_EQUALVERIFY instead of OP_EQUAL. All instances of OP_CHECKSIG within the `redeemScript` can then use the featureful set of SIGHASH flags.
 - OP_NEWCHECKSIG - Make another checksig opcode, possibly even using Schnorr signatures instead of standard secp256k1. 
