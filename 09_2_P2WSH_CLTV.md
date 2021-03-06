# 9.2: Script with CHECKLOCKTIMEVERIFY - Native Segwit P2WSH

> To follow along this tutorial and enter the commands step-by-step
> * Type `node` in a terminal after `cd` into `./code` for a Javascript prompt
> * Open the Bitcoin Core GUI console or use `bitcoin-cli` for the Bitcoin Core commands
> * Use `bx` aka `Libbitcoin-explorer` as a handy complement 

Let's create a native Segwit P2WSH transaction with a script that contains the `OP_CHECKLOCKTIMEVERIFY` absolute timelock opcode.

> Read more about OP_CHECKLOCKTIMEVERIFY in [BIP65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki)
> Read more about P2WSH in [BIP141 - Segregated Witness](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#p2wsh)


Here is the script.
Either Alice can redeem the output of the P2WSH after the timelock expiry (set 6 hours in the past), or Bob and Alice
can redeem the funds at any time. 
We will run both scenarios.
```javascript
function cltvCheckSigOutput (aQ, bQ, lockTime) {
  return bitcoin.script.compile([
    bitcoin.opcodes.OP_IF,
    bitcoin.script.number.encode(lockTime),
    bitcoin.opcodes.OP_CHECKLOCKTIMEVERIFY,
    bitcoin.opcodes.OP_DROP,

    bitcoin.opcodes.OP_ELSE,
    bQ.publicKey,
    bitcoin.opcodes.OP_CHECKSIGVERIFY,
    bitcoin.opcodes.OP_ENDIF,

    aQ.publicKey,
    bitcoin.opcodes.OP_CHECKSIG
  ])
}
```


## Creating and Funding the P2WSH 

Import libraries, test wallets and set the network and hashType.
```javascript
const bitcoin = require('bitcoinjs-lib')
const { alice, bob } = require('./wallets.json')
const network = bitcoin.networks.regtest
const hashType = bitcoin.Transaction.SIGHASH_ALL
```

We also need an additional library to help us with BIP65 absolute timelock encoding.
```javascript
const bip65 = require('bip65')
```

In both scenarios Alice_0 will get back the funds.
```javascript
const keyPairAlice0 = bitcoin.ECPair.fromWIF(alice[0].wif, network)
const p2wpkhAlice0 = bitcoin.payments.p2wpkh({pubkey: keyPairAlice0.publicKey, network})
```

Create a key pair for Bob_0.
```javascript
const keyPairBob0 = bitcoin.ECPair.fromWIF(bob[0].wif, network)
```

Encode the lockTime value according to BIP65 specification (now - 6 hours).
```javascript
const lockTime = bip65.encode({utc: Math.floor(Date.now() / 1000) - (3600 * 6)})
console.log('lockTime  ', lockTime)
```

Generate the witnessScript with CLTV. 
> In a P2WSH context, a redeem script is called a witness script.
> If you do it multiple times you will notice that the hex script is never the same, this is because of the timestamp.
```javascript
const witnessScript = cltvCheckSigOutput(keyPairAlice0, keyPairBob0, lockTime)
console.log('witnessScript  ', witnessScript.toString('hex'))
```

You can decode the script in Bitcoin Core CLI with `decodescript`.

Generate the P2WSH.
> If you do it multiple times you will notice that the P2WSH address is never the same, this is because of witnessScript.
```javascript
const p2wsh = bitcoin.payments.p2wsh({redeem: {output: witnessScript, network}, network})
console.log('P2WSH address  ', p2wsh.address)
```

Send 1 BTC to this P2WSH address.
```
$ sendtoaddress [p2wsh.address] 1
```

Get the output index so that we have the outpoint (txid / vout).
```
$ getrawtransaction "txid" true
```

The output of our funding transaction has a locking script composed of <00 version byte> + <32-byte hash witness program>.
SHA256 of the witnessScript must match the 32-byte witness program.
```javascript
bitcoin.crypto.sha256(witnessScript).toString('hex')
```


## Preparing the spending transaction

Now let's prepare the spending transaction by setting input and output, and the nLockTime value.

Create a BitcoinJS transaction builder object.
```javascript
const txb = new bitcoin.TransactionBuilder(network)
```

We need to set the transaction-level locktime in our redeem transaction in order to spend a CLTV.
You can use the same value as in the witnessScript.
> Because CLTV actually uses nLocktime enforcement consensus rules 
> the time is checked indirectly by comparing redeem transaction nLocktime with the CLTV value.
> nLocktime must be <= present time and >= CLTV timelock
```javascript
txb.setLockTime(lockTime)
```

Create the input by referencing the outpoint of our P2WSH funding transaction.
The input-level nSequence value needs to be change to `0xfffffffe`, which means that nSequence is disabled, nLocktime is 
enabled and RBF is not signaled.
```javascript
// txb.addInput(prevTx, input.vout, input.sequence, prevTxScript)
txb.addInput('TX_ID', TX_VOUT, 0xfffffffe)
```

Alice_0 will redeem the fund to her P2WPKH address, leaving 100 000 satoshis for the mining fees.
```javascript
txb.addOutput(p2wpkhAlice0.address, 999e5)
```

Prepare the transaction.
```javascript
const tx = txb.buildIncomplete()
```


## Adding the witness stack

Now we can update the transaction with the witness stack (`txinwitness` field), providing a solution to the locking script.

We generate the hash that will be used to produce the signatures.
> Note that we use a special method `hashForWitnessV0` for Segwit transactions.
```javascript
// hashForWitnessV0(inIndex, prevOutScript, value, hashType)
const signatureHash = tx.hashForWitnessV0(0, witnessScript, 1e8, hashType)
```

There are two ways to redeem the funds, either Alice after the timelock expiry or Alice and Bob at any time.
We control which branch of the script we want to run by ending our unlocking script with a boolean value.

First branch: {Alice's signature} OP_TRUE
```javascript
const witnessStackFirstBranch = bitcoin.payments.p2wsh({
  redeem: {
    input: bitcoin.script.compile([
      bitcoin.script.signature.encode(keyPairAlice0.sign(signatureHash), hashType),
      bitcoin.opcodes.OP_TRUE,
    ]),
    output: witnessScript
  }
}).witness

console.log('First branch witness stack  ', witnessStackFirstBranch.map(x => x.toString('hex')))
```

Second branch: {Alice's signature} {Bob's signature} OP_FALSE
```javascript
const witnessStackSecondBranch = bitcoin.payments.p2wsh({
  redeem: {
    input: bitcoin.script.compile([
      bitcoin.script.signature.encode(keyPairAlice0.sign(signatureHash), hashType),
      bitcoin.script.signature.encode(keyPairBob0.sign(signatureHash), hashType),
      bitcoin.opcodes.OP_FALSE
    ]),
    output: witnessScript
  }
}).witness

console.log('First branch witness stack  ', witnessStackSecondBranch.map(x => x.toString('hex')))
```

We provide the witness stack that BitcoinJS prepared for us. 
```javascript
tx.setWitness(0, [witnessStackFirstBranch OR witnessStackSecondBranch])
```

Get the raw hex serialization.
> No `build` step here as we have already called `buildIncomplete`
```javascript
console.log('tx.toHex  ', tx.toHex())
```

Inspect the raw transaction with Bitcoin Core CLI, check that everything is correct.
```
$ decoderawtransaction "hexstring"
```


## Broadcasting the transaction

> Beware of the `mediantime` of our node to be more than the timelock value.
> ```
> $ getblockchaininfo
> ```
> On regtest the mediantime is quite unpredictable. 
> So you just need to generate enough blocks, can't tell you how many. 
> ```
> $ generate 10
> ```

It's time to broadcast the transaction via Bitcoin Core CLI.
```
$ sendrawtransaction "hexstring"
```

Inspect the transaction.
```
$ getrawtransaction "txid" true
```


## Observations

For both scenarios we note that our scriptSig is empty.

For the first scenario, we note that our witness stack contains
  * Alice_0 signature
  * 1, which is equivalent to OP_TRUE
  * the witness script, that we can decode with `decodescript` 
  
For the second scenario, we note that our witness stack contains
  * Alice_0 signature
  * Bob_0 signature
  * an empty string, which is equivalent to OP_FALSE
  * the witness script, that we can decode with `decodescript`


## What's Next?

Continue "PART THREE: PAY TO SCRIPT HASH" with [9.3: Script with CHECKSEQUENCEVERIFY - Legacy P2SH](09_3_P2SH_CSV.md).
