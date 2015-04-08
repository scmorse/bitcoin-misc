## SIGHASH Proposal

This document is the result of an IRC discussion from 4/7/15 with gmaxwell, tdryja, and StephenM347, on how to enable the bitcoin lightning network with more fully capable set of SIGHASH flags.

The current set of SIGHASH parameters are defined by the [below code snippet](https://github.com/bitcoin/bitcoin/blob/v0.10.0/src/script/interpreter.h#L21-L28):

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

However, these could be extended to provide greater flexebility. In particular, for the bitcoin lightning network, it would be useful to sign transactions without serializing in the TXID. In addition, it would be useful both for the lightning network and for hardware wallets to include the value of the UTXO being spent in the serialized transaction that is hashed for signing, as was proposed in [this bitcointalk thread](https://bitcointalk.org/index.php?topic=181734.0). 

I propose using a 4-byte SIGHASH flag. One bytes determines the inclusion/exclusion of data at the input for the currently executing `scriptSig` (and the output at the same index, if such an output exists). One more byte determines what data from other inputs/outputs should be serialized for the hash to sign. The remaining space determines what global transactions specific fields should be serialized for the hash to sign. In addition, there is a flag for signing a 32 byte value that hash been pushed on the stack. Using these fields, it is completely customizable which transaction data becomes hashed for the signature hash. These should fully enable any seen or unforseen usecase of the signature serializer.

The new SIGHASH flags are described below.

```
/** Signature hash types/flags */
enum
{
    // Input Specific
    SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY     = 0x01,
    SIGHASH_WITHOUT_PREV_VALUE            = 0x02,
    SIGHASH_WITHOUT_INPUT_TXID            = 0x04,
    SIGHASH_WITHOUT_INPUT_INDEX           = 0x08,
    SIGHASH_WITHOUT_INPUT_SCRIPTCODE      = 0x10, /* scriptSig, minus signature itself, and taking into account OP_CODESEPARATOR */
    SIGHASH_WITHOUT_INPUT_SEQUENCE        = 0x20,
    
    // Output Specific
    SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY   = 0x40,
    SIGHASH_WITHOUT_OUTPUT_VALUE          = 0x80,
    
    // Transaction Specific
    SIGHASH_WITHOUT_TX_VERSION            = 0x100000,
    SIGHASH_WITHOUT_TX_LOCKTIME           = 0x200000,
    
    // Sign value not derived from transaction
    // (Whenever nHashType is negative, the script 
    SIGHASH_SIGN_STACK_ELEMENT            = 0x10000000,
};
```

For example, With this method, SIHASH_ALL is defined by:

```
int SIGHASH_ALL = 
```

To be soft-fork compatible, obviously these changes need to be done through an alternative method to using the single byte nHashType for OP_CHECKSIG. Two such alternatives are:

 - P3SH - Use a scriptPubKey similar to P2SH, but ending with OP_EQUALVERIFY instead of OP_EQUAL. All instances of OP_CHECKSIG within the `redeemScript` can then use the featureful set of SIGHASH flags.
 - OP_NEWCHECKSIG - Make another checksig opcode, possibly even using Schnorr signatures instead of standard secp256k1. 
