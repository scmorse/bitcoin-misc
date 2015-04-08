## Build your own nHashType 

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

I propose using a 4-byte nHashType. One byte determines the inclusion/exclusion of data at the input for the currently executing `scriptSig` (and the output at the same index, if such an output exists). One more byte determines what data from other inputs/outputs should be serialized for the hash to sign. The remaining space determines what global transaction specific fields should be serialized for the hash to sign. In addition, there is a flag for signing a 32 byte value that hash been pushed on the stack. Using these fields, it is completely customizable which transaction data becomes hashed for the signature hash. These should fully enable any seen or unforseen use case of the CTransactionSignatureSerializer.

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
    SIGHASH_WITHOUT_INPUT_SEQUENCE        = 0x10,

    // Output Specific
    SIGHASH_WITHOUT_OUTPUT_SCRIPTPUBKEY   = 0x20,
    SIGHASH_WITHOUT_OUTPUT_VALUE          = 0x40,

    // Whether to serialize the other (other than self) inputs/outputs at all
    SIGHASH_WITHOUT_INPUTS                = 0x010000,
    SIGHASH_WITHOUT_OUTPUTS               = 0x020000,

    // Whether to serialize this input/output at all (these take priority over SIGHASH_WITHOUT_INPUTS and SIGHASH_WITHOUT_OUTPUTS)
    SIGHASH_WITHOUT_INPUT_SELF            = 0x040000,
    SIGHASH_WITHOUT_OUTPUT_SELF           = 0x080000,

    // Transaction specific fields
    SIGHASH_WITHOUT_TX_VERSION            = 0x100000,
    SIGHASH_WITHOUT_TX_LOCKTIME           = 0x200000,


    // Sign value not derived from transaction
    // (Whenever nHashType is negative, the script signature is for the value on the stack, e.g. stacktop(-3))
    SIGHASH_SIGN_STACK_ELEMENT            = 0x10000000,
};
```

Generally, more significant bits are given to flags with higher priority.

Note, to limit the data required to calculate a signature Hash, SIGHASH_WITHOUT_PREV_SCRIPTPUBKEY and SIGHASH_WITHOUT_PREV_VALUE are assumed true on all other inputs (other than the currently executing).

For example, with this method, SIGHASH_ALL, SIGHASH_SINGLE, and SIGHASH_NONE are currently defined by:

```
int SIGHASH_ALL = SIGHASH_WITHOUT_PREV_VALUE;

int nAtIndex = SIGHASH_WITHOUT_PREV_VALUE;
int nAtOther = SIGHASH_WITHOUT_OUTPUTS | SIGHASH_WITHOUT_OUTPUT_SELF;
int SIGHASH_NONE = (nAtIndex << 8) | nAtOther;

int nAtIndex = SIGHASH_WITHOUT_PREV_VALUE;
int nAtOther = SIGHASH_WITHOUT_OUTPUTS | SIGHASH_WITHOUT_INPUTS;
int SIGHASH_SINGLE = (nAtIndex << 8) | nAtOther;
```

To make a flag also use SIGHASH_ANYONECANPAY:

```
// nHashType already defined, want to make it SIGHASH_ANYONECANPAY
nHashType = nHashType & (~SIGHASH_WITHOUT_INPUT_SELF) | (SIGHASH_WITHOUT_INPUTS);
```
-----

Very rough draft implementation: 

```
/**
 * Wrapper that serializes like CTransaction, but with the modifications
 *  required for the signature hash done in-place
 */
class CTransactionSignatureSerializer {
private:
    const CTransaction &txTo;  //! reference to the spending transaction (the one being serialized)
    const CScript &scriptCode; //! output script being consumed
    const unsigned int nIn;    //! input index of txTo being signed
    const int nHashType;       //! nHashType determines logic of serialization

public:
    CTransactionSignatureSerializer(const CTransaction &txToIn, const CScript &scriptCodeIn, unsigned int nInIn, int nHashTypeIn) :
        txTo(txToIn), scriptCode(scriptCodeIn), nIn(nInIn), nHashType(nHashTypeIn) {}

    /** Serialize the passed scriptCode, skipping OP_CODESEPARATORs */
    template<typename S>
    void SerializeScriptCode(S &s, int nType, int nVersion) const {
        CScript::const_iterator it = scriptCode.begin();
        CScript::const_iterator itBegin = it;
        opcodetype opcode;
        unsigned int nCodeSeparators = 0;
        while (scriptCode.GetOp(it, opcode)) {
            if (opcode == OP_CODESEPARATOR)
                nCodeSeparators++;
        }
        ::WriteCompactSize(s, scriptCode.size() - nCodeSeparators);
        it = itBegin;
        while (scriptCode.GetOp(it, opcode)) {
            if (opcode == OP_CODESEPARATOR) {
                s.write((char*)&itBegin[0], it-itBegin-1);
                itBegin = it;
            }
        }
        if (itBegin != scriptCode.end())
            s.write((char*)&itBegin[0], it-itBegin);
    }

    /** Serialize an input of txTo */
    template<typename S>
    void SerializeInput(S &s, unsigned int nInput, int nType, int nVersion) const {
        // Serialize the prevout
        if (!())
            ::Serialize(s, txTo.vin[nInput].prevout, nType, nVersion);
        // Serialize the script
        if (nInput != nIn)
            // Blank out other inputs' signatures
            ::Serialize(s, CScript(), nType, nVersion);
        else
            SerializeScriptCode(s, nType, nVersion);
        // Serialize the nSequence
        if (nInput != nIn && (fHashSingle || fHashNone))
            // let the others update at will
            ::Serialize(s, (int)0, nType, nVersion);
        else
            ::Serialize(s, txTo.vin[nInput].nSequence, nType, nVersion);
    }

    /** Serialize an output of txTo */
    template<typename S>
    void SerializeOutput(S &s, unsigned int nOutput, int nType, int nVersion) const {
        if (fHashSingle && nOutput != nIn)
            // Do not lock-in the txout payee at other indices as txin
            ::Serialize(s, CTxOut(), nType, nVersion);
        else
            ::Serialize(s, txTo.vout[nOutput], nType, nVersion);
    }

    /** Serialize txTo */
    template<typename S>
    void Serialize(S &s, int nType, int nVersion) const {
        // Serialize nVersion
        ::Serialize(s, txTo.nVersion, nType, nVersion);
        // Serialize vin
        unsigned int nInputs = fAnyoneCanPay ? 1 : txTo.vin.size();
        ::WriteCompactSize(s, nInputs);
        for (unsigned int nInput = 0; nInput < nInputs; nInput++)
             SerializeInput(s, nInput, nType, nVersion);
        // Serialize vout
        unsigned int nOutputs = fHashNone ? 0 : (fHashSingle ? nIn+1 : txTo.vout.size());
        ::WriteCompactSize(s, nOutputs);
        for (unsigned int nOutput = 0; nOutput < nOutputs; nOutput++)
             SerializeOutput(s, nOutput, nType, nVersion);
        // Serialize nLockTime
        ::Serialize(s, txTo.nLockTime, nType, nVersion);
    }
};
```

-----

To be soft-fork compatible, obviously these changes need to be done through an alternate method to using the single byte nHashType with OP_CHECKSIG. Two such alternatives are:

 - P3SH - Use a scriptPubKey similar to P2SH, but ending with OP_EQUALVERIFY instead of OP_EQUAL. All instances of OP_CHECKSIG within the `redeemScript` can then use the featureful set of SIGHASH flags.
 - OP_NEWCHECKSIG - Make another checksig opcode, possibly even using Schnorr signatures instead of standard secp256k1.

Finally, while it may seem unnecessary to use an extra 4 bytes per signature, it should be noted that since this is in the scriptSig, it is entirely prunable. 

This may be too complex since we know the use cases that are in mind for the sighash flags. However, there may be unforeseen use cases, and this soft-fork will enable all possible sighash flag configurations, rather than needing to soft fork every time an alternate sighash flag is needed.
