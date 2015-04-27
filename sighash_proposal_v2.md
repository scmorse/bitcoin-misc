## Build your own nHashType 

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

Instead of the standard 1-byte sighash flag, we may use a 2-byte nHashType that offers much more flexibility. Seven bits determine the inclusion/exclusion of data at the input for the currently executing `scriptSig` (and the output at the same index, if such an output exists). Five other bits determine what data from other inputs/outputs should be serialized for the hash to sign. Two bits determine what global transaction specific fields should be serialized for the hash to sign. The remaining two bits are unused. Using these fields, it is completely customizable which transaction data becomes hashed for the signature hash. These should fully enable any seen or unforseen use case of the CTransactionSignatureSerializer.

The new SIGHASH flags are below.

```
/** Signature hash flags */
enum
{
    // Input Specific
    SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY     = 0x0001,
    SIGHASH_WITHOUT_PREV_VALUE            = 0x0002,
    SIGHASH_WITHOUT_INPUT_TXID            = 0x0004,
    SIGHASH_WITHOUT_INPUT_INDEX           = 0x0008,
    SIGHASH_WITHOUT_INPUT_SEQUENCE        = 0x0010,

    // Output Specific
    SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY   = 0x0020,
    SIGHASH_WITHOUT_OUTPUT_VALUE          = 0x0040,
    
    // Reserved for using the above flags at the currently executing index
    //                                      0x0080
    //                                      0x0100
    //                                      0x0200
    //                                      0x0400
    //                                      0x0800
    //                                      0x1000
    //                                      0x2000

    // Transaction specific fields
    SIGHASH_WITHOUT_TX_LOCKTIME           = 0x4000,
    SIGHASH_WITHOUT_TX_VERSION            = 0x8000,
};
```

Generally, more significant bits are given to flags with higher priority.

Note, to limit the data required to calculate a signature Hash, SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY and SIGHASH_WITHOUT_PREV_VALUE are assumed true on all other inputs (other than the currently executing). In practice, this means the two least significant bits of the nHashType short are unused. They should be defaulted to 1. To prevent malleability, a script executing this without those bits set to 1 should mark the transaction invalid. 

For example, with these flags, SIGHASH_ALL, SIGHASH_SINGLE, and SIGHASH_NONE, as they exist today, would be equivalent to:

```
int nAtIndex = SIGHASH_WITHOUT_PREV_VALUE;
int nAtOther = SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY | 
               SIGHASH_WITHOUT_PREV_VALUE;
int SIGHASH_ALL = (nAtIndex << 7) | nAtOther;

/////////////////////////////////////////////

int nAtIndex = SIGHASH_WITHOUT_PREV_VALUE |
               SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY |
               SIGHASH_WITHOUT_OUTPUT_VALUE;
int nAtOther = SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY | 
               SIGHASH_WITHOUT_PREV_VALUE | 
               SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY |
               SIGHASH_WITHOUT_OUTPUT_VALUE;
int SIGHASH_NONE = (nAtIndex << 7) | nAtOther;

/////////////////////////////////////////////

int nAtIndex = SIGHASH_WITHOUT_PREV_VALUE;
int nAtOther = SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY | 
               SIGHASH_WITHOUT_PREV_VALUE | 
               SIGHASH_WITHOUT_INPUT_TXID |
               SIGHASH_WITHOUT_INPUT_INDEX |
               SIGHASH_WITHOUT_INPUT_SEQUENCE |
               SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY |
               SIGHASH_WITHOUT_OUTPUT_VALUE;
int SIGHASH_SINGLE = (nAtIndex << 7) | nAtOther;
```

In this example, it is necessary to have SIGHASH_WITHOUT_PREV_VALUE in each set of input flags (`nAtIndex`) because, currently, the prevout's value is not serialized into the transaction. This would change, by default, so that the equivalent of [SIGHASH_WITHINPUTVALUE](https://bitcointalk.org/index.php?topic=181734.0) is the norm. To make a flag also use SIGHASH_ANYONECANPAY:

```
// nHashType already defined, want to make it use SIGHASH_ANYONECANPAY
int nAtOther = SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY | 
               SIGHASH_WITHOUT_PREV_VALUE | 
               SIGHASH_WITHOUT_INPUT_TXID |
               SIGHASH_WITHOUT_INPUT_INDEX |
               SIGHASH_WITHOUT_INPUT_SEQUENCE
nHashType = nHashType | nAtOther;
```

Specifying the three SIGHASH_WITHOUT_INPUT_\* flags simply specifies that those inputs should not be serialized at all, nor counted towards the number of inputs which is included in the serialization.

-----

These flags give a conceptual representation of what data is to be serialized, but they do not show the order or the method by which the transaction data is serialized. Part of this proposal is using a new order for serialization that makes optimizations for verifying signatures possible. In essence, the data that changes from input to input is serialized at the end of the transaction so that the midstate may be saved and reused at other points in the transaction. This does not prevent CPU exhaustion attacks as described [here](https://bitcointalk.org/?topic=140078), but it optimizes the speed at which transaction signature hashes can be evaluated.

Very rough draft implementation: 

```
/**
 * Wrapper that serializes like CTransaction, but with the modifications
 * required for completely configurable signature hash done in-place.
 */
class CConfigurableTransactionSignatureSerializer : CBaseTransactionSignatureSerializer {
private:
    const CAmount nValue;      //! nValue of ouptut being consumed
    const unsigned int nIn;    //! input index of txTo being signed
    const int nHashType;       //! nHashType determines logic of serialization

public:
    CConfigurableTransactionSignatureSerializer(const CTransaction &txToIn, const CScript &scriptCodeIn, CAmount nValueIn, unsigned int nInIn, int nHashTypeIn) :
        txTo(txToIn), scriptCode(scriptCodeIn), nValue(nValueIn), nIn(nInIn), nHashType(nHashTypeIn) {
        assert((nHashType & SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY) != 0);
        assert((nHashType & SIGHASH_WITHOUT_PREV_VALUE) != 0);
    }

    int PartialHashType(bool fCurrIndex) {
        // Assume SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY and SIGHASH_WITHOUT_PREV_VALUE true for other inputs
        return fCurrIndex ? (nHashType >> 7) : (nHashType);
    }

    /** Serialize an input of txTo */
    template<typename S>
    void SerializeInput(S &s, unsigned int nInput, bool fCurrIndex, int nType, int nVersion) const {
        int currHashType = PartialHashType(fCurrIndex);

        // Serialize the prevout
        if (!(currHashType & SIGHASH_WITHOUT_INPUT_TXID))
            ::Serialize(s, txTo.vin[nInput].prevout.hash, nType, nVersion);

        if (!(currHashType & SIGHASH_WITHOUT_INPUT_INDEX))
            ::Serialize(s, txTo.vin[nInput].prevout.n, nType, nVersion);

        if (!(currHashType & SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY))
            SerializeScriptCode(s, nType, nVersion);
        else
            // Blank out other inputs' signatures
            ::Serialize(s, CScript(), nType, nVersion);

        if (!(currHashType & SIGHASH_WITHOUT_PREV_VALUE))
            ::Serialize(s, nValue, nType, nVersion);

        if (!(currHashType & SIGHASH_WITHOUT_INPUT_SEQUENCE))
            ::Serialize(s, txTo.vin[nInput].nSequence, nType, nVersion);
    }

    /** Serialize an output of txTo */
    template<typename S>
    void SerializeOutput(S &s, unsigned int nOutput, bool fCurrIndex, int nType, int nVersion) const {
        int currHashType = PartialHashType(fCurrIndex);

        if (!(currHashType & SIGHASH_WITHOUT_OUTPUT_VALUE))
            ::Serialize(s, txTo.vout[nOutput].nValue, nType, nVersion);

        if (!(currHashType & SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY))
            ::Serialize(s, txTo.vout[nOutput].scriptPubKey, nType, nVersion);
        else
            ::Serialize(s, CScript(), nType, nVersion);
    }

    /** Serialize txTo */
    template<typename S>
    void Serialize(S &s, int nType, int nVersion) const {
        // From most significant flags to least significant

        int nAtIndex = PartialHashType(nIn);
        int nAtOther = PartialHashType(nIn+1);

        // Serialize nVersion
        if (!(nHashType & SIGHASH_WITHOUT_TX_VERSION))
            ::Serialize(s, txTo.nVersion, nType, nVersion);

        // Serialize nLockTime - TODO maybe this should actually go near end?
        if (!(nHashType & SIGHASH_WITHOUT_TX_LOCKTIME))
            ::Serialize(s, txTo.nLockTime, nType, nVersion);

        // Serialize 'other' outputs
        unsigned int nOutputs = ((nAtOther & SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY) == 0) ||
                                ((nAtOther & SIGHASH_WITHOUT_OUTPUT_VALUE)        == 0) ? txTo.vout.size() : 0;
        ::WriteCompactSize(s, nOutputs);

        for (unsigned int nOutput = 0; nOutput < nOutputs; nOutput++) {
            SerializeOutput(s, nOutput, false, nType, nVersion);
        }

        // Serialize 'other' inputs
        unsigned int nInputs = ((nAtOther & SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY) == 0) ||
                               ((nAtOther & SIGHASH_WITHOUT_PREV_VALUE)        == 0) ||
                               ((nAtOther & SIGHASH_WITHOUT_INPUT_TXID)        == 0) ||
                               ((nAtOther & SIGHASH_WITHOUT_INPUT_INDEX)       == 0) ||
                               ((nAtOther & SIGHASH_WITHOUT_INPUT_SEQUENCE)    == 0) ? txTo.vin.size() : 0;
        ::WriteCompactSize(s, nInputs);

        for (unsigned int nInput = 0; nInput < nInputs; nInput++) {
            SerializeInput(s, nInput, false, nType, nVersion);
        }

        // TODO take snapshot for common sighash types at this point

        // Serialize 'this' output, if the output exists and the sighash type doesn't completely remove it
        bool fSerializeOutput = nIn < txTo.vout.size() && 
                                (((nAtIndex & SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY) == 0) ||
                                 ((nAtIndex & SIGHASH_WITHOUT_OUTPUT_VALUE)        == 0));
        if (fSerializeOutput) {
            ::WriteCompactSize(s, 1);
            SerializeOutput(s, nIn, true, nType, nVersion);
        }

        // Serialize 'this' input, if the sighash type doesn't completely remove it
        bool fSerializeInput = ((nAtIndex & SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY) == 0) ||
                               ((nAtIndex & SIGHASH_WITHOUT_PREV_VALUE)        == 0) ||
                               ((nAtIndex & SIGHASH_WITHOUT_INPUT_TXID)        == 0) ||
                               ((nAtIndex & SIGHASH_WITHOUT_INPUT_INDEX)       == 0) ||
                               ((nAtIndex & SIGHASH_WITHOUT_INPUT_SEQUENCE)    == 0);
        if (fSerializeInput) {
            ::WriteCompactSize(s, 1);
            SerializeOutput(s, nIn, true, nType, nVersion);
        }
        
    }
};
```

-----

To be soft-fork compatible, obviously these changes need to be done through an alternate method to using the single byte nHashType with OP_CHECKSIG. Two such alternatives are:

 - P3SH - There are a few variations on this, but it is mainly a variant of P2SH. My favorite is to use a `scriptPubKey` with the following format:

```
OP_HASH160 0x14 {data} OP_EQUALVERIFY OP_3
```

 - OP_NEWCHECKSIG - Make another checksig opcode, possibly even using Schnorr signatures instead of standard secp256k1.

This may be too complex since we know the use cases that are in mind for the sighash flags. However, there may be unforeseen use cases, and this soft-fork will enable all possible sighash flag configurations, rather than needing to soft fork every time an alternate sighash flag is needed. In addition, it permits the use of an optimized SignatureHash() function that can avoid re-hashing the same transaction data many many times to verify inputs.

------

Change log:

I removed the sighash flag for signing a value on the stack because it seemed to be the wrong place for such an operation. Using an OP_NOP to create a modified OP_CHECKDATASIG seemed more appropriate. The current OP_CHECKSIG is really a OP_CHECKTXSIG and shouldn't be used to check sigs for data.

I also removed the ability to not sign the input that the signature is being created for. If someone can think of a reason to add this back in then I'd be interested, but it seems to me that the whole point of creating an input is to sign that input as a valid spend of the prevout to fund the transaction.

SIGHASH_WITHOUT_OUTPUT_SELF was removed because simply flagging both SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY and SIGHASH_WITHOUT_OUTPUT_VALUE can have the same effect, without needing to provide a check that the transaction was serialized with the output but without any of the outputs data. Similarly for SIGHASH_WITHOUT_INPUTS and SIGHASH_WITHOUT_OUTPUTS.
