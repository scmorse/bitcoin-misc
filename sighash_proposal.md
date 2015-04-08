## SIGHASH Proposal

This document is the result of an IRC discussion from 4/7/15 with gmaxwell, tdryja, and StephenM347, on how to enable the bitcoin lightning network with a more fully featured set of SIGHASH flags.

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

These cover the main uses cases. However, these could be extended to provide greater flexibility for future use cases. In particular, for the bitcoin lightning network, it would be useful to sign transactions without serializing in the TXID. In addition, it would be useful both for the lightning network and for hardware wallets (and, arguably, anyone) to include the value of the UTXO being spent in the serialized transaction that is hashed for signing, as was proposed in [this bitcointalk thread](https://bitcointalk.org/index.php?topic=181734.0). 

I propose using a 4-byte SIGHASH flag. One bytes determines the inclusion/exclusion of data at the input for the currently executing `scriptSig` (and the output at the same index, if such an output exists). One more byte determines what data from other inputs/outputs should be serialized for the hash to sign. The remaining space determines what global transaction specific fields should be serialized for the hash to sign. In addition, there is a flag for signing a 32 byte value that hash been pushed on the stack. Using these fields, it is completely customizable which transaction data becomes hashed for the signature hash. These should fully enable any seen or unforseen use case of the CTransactionSignatureSerializer.

The new SIGHASH flags are below.

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
    SIGHASH_WITHOUT_INPUT                 = 0x010000,
    SIGHASH_WITHOUT_OUTPUT                = 0x020000,
    SIGHASH_WITHOUT_OTHER_INPUTS          = 0x040000,
    SIGHASH_WITHOUT_OTHER_OUTPUTS         = 0x080000,
    SIGHASH_WITHOUT_TX_VERSION            = 0x100000,
    SIGHASH_WITHOUT_TX_LOCKTIME           = 0x200000,

    
    // Sign value not derived from transaction
    // (Whenever nHashType is negative, the script signature is for the value on the stack, e.g. stacktop(-3))
    SIGHASH_SIGN_STACK_ELEMENT            = 0x10000000,
};
```

Note, to limit the data required to calculate a signature Hash, SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY and SIGHASH_WITHOUT_PREV_VALUE are assumed true on all other inputs (other than the currently executing).

For example, with this method, SIGHASH_ALL is currently defined by:

```
int nAtIndex = SIGHASH_WITHOUT_PREV_VALUE;
int nAtOther = SIGHASH_WITHOUT_INPUT_SCRIPTCODE;
int SIGHASH_ALL = (nAtOther << 8) | nAtIndex;
```

To be soft-fork compatible, obviously these changes need to be done through an alternate method to using the single byte nHashType with OP_CHECKSIG. Two such alternatives are:

 - P3SH - Use a scriptPubKey similar to P2SH, but ending with OP_EQUALVERIFY instead of OP_EQUAL. All instances of OP_CHECKSIG within the `redeemScript` can then use the featureful set of SIGHASH flags.
 - OP_NEWCHECKSIG - Make another checksig opcode, possibly even using Schnorr signatures instead of standard secp256k1. 

It may also be preferable to put nHashType on the stack instead of serialized into the signature.

Finally, while it may seem unnecessary to use an extra 4 bytes per signature, it should be noted that since this is in the scriptSig, it is entirely prunable. 

This may be too complex since we know the use cases that are in mind for the sighash flags. However, there may be unforeseen use cases, and this soft-fork will enable all possible sighash flag configurations, rather than needing to soft fork every time an alternate sighash flag is needed.
