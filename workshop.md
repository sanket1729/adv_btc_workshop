Advancing bitcoin Hands on Miniscript workshop

### Pre-requities:

- rust: Install using `rustup`. We will use rust 1.58 (stable as of March 4 2022)

- hal:
	- You can clone via https `https://github.com/sanket1729/hal`
	- Checkout the `adv_btc_workshop` branch: `git checkout adv_btc_workshop`.
	- Build the project using `cargo build`.
	- Optionally, for your convenience you can add `./target/debug/hal` to your `$PATH` in the current session.
- Bitcoin core(optional, but recommended):
	- go to `https://github.com/bitcoin/bitcoin`
	- Read the instructions in `doc` to install on your operating system.
	- We don't need the sync in for this workshop. We are only going to use bitcoin core in regtest
- jq: For parsing json.


### Miniscript Introduction:

- Miniscript is a language for writing (a subset of) Bitcoin Scripts in a structured way, enabling analysis, composition, generic signing and more. Bitcoin Script is an unusual stack-based language with many edge cases, designed for implementing spending conditions consisting of various combinations of signatures, hash locks, and time locks. Yet despite being limited in functionality it is still highly nontrivial to:

- Given a combination of spending conditions, finding the most economical script to implement it.
- Given two scripts, construct a script that implements a composition of their spending conditions (e.g. a multisig where one of the "keys" is another multisig).
- Given a script, find out what spending conditions it permits.
- Given a script and access to a sufficient set of private keys, construct a general satisfying witness for it.
- Given a script, be able to predict the cost of spending an output.
- Given a script, know whether particular resource limitations like the ops limit might be hit when spending.

For everything miniscript, `bitcoin.sipa.be/miniscript`

### What is Psbt BIP174?

- Transaction signers need more information other just the current transaction and private keys in order to satisfy it. This information might include
	- Key derivation paths/origin information
	- Spending utxo in order to compute the signature hash message
	- Witness scripts to understand the script we are spending.
- You can read more about Psbts at (link to BIP174/bip370/bip371)
- Pbsts already have support for satisfaction of signatures and timelocks, but the hashlocks require new psbt key value pairs.


### What are output descriptors?

Output Script Descriptors are a simple language which can be used to describe collections of output scripts. See BIP380 for details.

- Miniscript would be added as new `SCRIPT` to `wsh` and `tr`

### Policy language:

Simplest form in which we can spending conditions. Abstract form for representing spending conditions
- Keys
- HashLocks
- Timelocks
- ands, ors, thresholds of the above

### Spending a taproot Miniscript Descriptor

In this workshop, we will create a complex taproot descriptor using `hal` and try to spend it.

## Creating a taproot descriptor

- Decide your spending policy. What policy should govern the funds? To follow along with this workshop consider the following policy:
- To follow along this workshop, you can use this spending policy.
```
- 99@thresh(2, pk(hA), pk(S)),
- 1@or
	- 99@pk(cA)
	- 1@(and(pk(In), older(LONG_TIME)) # this indicates the key will never likely be used to spend
```
There are three different
- Happy case that we expect to happen occur 99% of times
	- Our hot mobile wallet key `hA`
	- Co-signer key `S` that will sign the transaction. This could be third-party like Green that provides 2FA service
- (1%) If we lose mobile wallet, or co-signer is not participating, or if we forget the OTP:
	- We can use cold keys `Ca` and `Cb` from paper wallets/HW wallets etc
- (0.01%)If we just happen to die :( , we have already given a paper wallet with pk inheritance `In` to trusted family/friend that they can use after a long time `LONG_TIME`. We can keep spending funds back to policy to reset the `LONG_TIME`.

our final policy is then
```
POL="or(99@thresh(2,pk(hA),pk(S)),1@or(99@pk(Ca),1@and(pk(In),older(9))))"
```

Compile the policy to a descriptor using `hal ms policy $POL`.

```
DESC="tr(Ca,{and_v(v:pk(In),older(9)),multi_a(2,hA,S)})"
```
If you an experienced bitcoin dev and have dealt with psbts before, you can create your own spending policy. Perhaps, even collaborate with your neighbor in creating a policy that in such a way that both of you can spend funds. Don't worry if you mess up, regtest coins are free!

### Sample new keys with hal

For this workshop, we will directly sample private keys from hal and use it create descriptor. Sample keys using `hal key generate`
```
# My secret insecure Wallet
hA = L5T1XNNoMxQpDyQGD8dgMvEB9wpxScxcovaYAmJhr1zCgqcCCd9v
S  = Kwc3gGyaLJe6HvZWTtyS8eu49assTZwdvGfcXKVke7tZjYmb6uJr
In = KzCB2QwrKen2Bnx2Q299iPUV8TF1UzPGWDGZFtpUtnodeegSCn9F
cA = KyyvHq1wxvVEi3V8MfoqaC8WXkcqpiNkr9BQJocKdRLJDAa6LEVB
```
Replace the keys with private keys and save the descriptor

```
PRIV_DESC="tr(KyyvHq1wxvVEi3V8MfoqaC8WXkcqpiNkr9BQJocKdRLJDAa6LEVB,{and_v(v:pk(KzCB2QwrKen2Bnx2Q299iPUV8TF1UzPGWDGZFtpUtnodeegSCn9F),older(9)),multi_a(2,L5T1XNNoMxQpDyQGD8dgMvEB9wpxScxcovaYAmJhr1zCgqcCCd9v,Kwc3gGyaLJe6HvZWTtyS8eu49assTZwdvGfcXKVke7tZjYmb6uJr)})"
```

Get the information about the descriptor using `hal ms descriptor --regtest $PRIV_DESC`. This will output a `key_map` and a descriptor replacing all the private keys with public keys and a public descriptor. Copy paste the key map in a file and this is your INSECURE wallet! For this demo, we only deal with stateless wallets spending a single transaction. In practice, these would be BIP32 wallets with xpubs and some derivation paths with wildcards.

```
"key_map": {
	"035ef213d38b03f53da9130be99e77c02452e2a292d31a71b2c7af4436e959eca1": "KyyvHq1wxvVEi3V8MfoqaC8WXkcqpiNkr9BQJocKdRLJDAa6LEVB",
	"02194daca75bf1b3fcab8904f33ba2c32b346789f17fabf9eade31d996894343a5": "L5T1XNNoMxQpDyQGD8dgMvEB9wpxScxcovaYAmJhr1zCgqcCCd9v",
	"0280e104a49f9638684083feee1ac61177b3d2f3c7748f690013a13d6ef23cc000": "Kwc3gGyaLJe6HvZWTtyS8eu49assTZwdvGfcXKVke7tZjYmb6uJr",
	"025eba9784b8e52c6ec36961c055ac026ddd241f0efc954ced67a8eb1596bebdf8": "KzCB2QwrKen2Bnx2Q299iPUV8TF1UzPGWDGZFtpUtnodeegSCn9F"
}
```

```
DESC_PUB=$(hal ms descriptor $PRIV_DESC | jq '.desc_pub' | tr -d '"')
```

Inspect the policy and see if it matches your expectation.

Save the address:
```
ADDR=bcrt1pzk99wda22uprx5exsnqzlgdvup2pqsu32lys2v8x649r49qeh5gsy9lcjn
```

Using bitcoin-core client send 1 BTC to this address. If you don't have bitcoin core installed, you can skip this step. We will use a hardcoded prevout instead. In the `src` folder in bitcoin core directory, you can run the following commands to get the rawtx that spends funds to this transaction. You would need to generate blocks to get initial balance using `./bitcoin-cli -generate 102`
```
TX=$(./bitcoin-cli --regtest sendtoaddress $ADDR 1.0)
RAW_TX=$(./bitcoin-cli --regtest getrawtransaction $TX)
```

## Spending the fund tx

First, let us create a rawtransaction that spends from this address using `hal tx create`. You can also use `bitcoin-cli` to create tx instead. If you don't have access to `bitcoin-cli`, you can use any random `prevout` in `inputs[0]` instead of in spending the input. If you are using `bitcoin-cli`, lookup the correct prevout by inspecting `hal tx decode $RAW_TX`. (Again, you can also use `bitcoin-cli` here).

```
{
  "version": 2,
  "locktime": 0,
  "inputs": [
    {
      "prevout": "aa75baf715f173e92bb1ca232462d280b8ce431ad8a303c30b49fcf70686d4c9:1",
      "sequence": 4294967295
    }
  ],
  "outputs": [
    {
      "value": 99990000,
      "script_pub_key": {
        "hex": "a91405394a3a5dedce4f945ed9f650fa9ff23f011d4687"
      }
    }
  ]
}
```

Save the spending raw_tx in a variable `SPEND_TX`. You can check this has input and one output by inspecting `hal tx decode $SPEND_TX`

### Tx signing and finalizing with psbt

First create a psbt with `hal psbt create`. At any point, you can inspect the psbt using `hal psbt decode`.

```
RAW_PSBT=$(hal psbt create $SPEND_TX)
```

Next we need to address information about the witness-utxo that we are spending. Again, using hal we will inspect the transaction to get information about `PSBT_INPUT_WITNESS_UTXO` to create a spending transaction that spends this tx. Inspect the `witness_utxo` field in `hal tx decode $RAW_TX`. If you have `jq`, you can find the spending witness_utxo as
```
WITNESS_UTXO=$(hal tx decode $RAW_TX | jq '.outputs[idx].witness_utxo' | tr -d '"')
```
Replace the `[idx]` with spending outpoint index. Next update the psbt with spending information.

```
UNUPDATED_PSBT=$(hal psbt edit $RAW_PSBT --input-idx=0 --witness-utxo=$WITNESS_UTXO)
UNSIGNED_PSBT=$(hal psbt utxoupdate $UNUPDATED_PSBT 0 $DESC_PUB)
```

### Signing with script spend path: Using keys Ha and cosigner S.

Compute the signed Psbts

```
HA_SIGNED=$(hal psbt rawsign $UNSIGNED_PSBT 0 L5T1XNNoMxQpDyQGD8dgMvEB9wpxScxcovaYAmJhr1zCgqcCCd9v)
S_SIGNED=$(hal psbt rawsign $UNSIGNED_PSBT 0 Kwc3gGyaLJe6HvZWTtyS8eu49assTZwdvGfcXKVke7tZjYmb6uJr)
```

Combine the signed psbts with signatures from both parties

```
PSBT_MERGED=$(hal psbt merge $HA_SIGNED $S_SIGNED)
```

Finalize the psbt, the miniscript finalizer will figure out what to based on available sigs, hash preimages

```
EXT_TX2=$(hal psbt finalize $PSBT_MERGED)
```

### Signing with key spend path: Cold wallet

Next, we will sign the psbt. While signing, we can choose which paths we need to spend with. First, let us try the simplest spend path, the key spend path. Note that this corresponds to case where we have the cold key Ca.

```
KEY_SIGNED=$(hal psbt rawsign $UNSIGNED_PSBT 0 KyyvHq1wxvVEi3V8MfoqaC8WXkcqpiNkr9BQJocKdRLJDAa6LEVB)
EXT_TX2=$(hal psbt finalize $KEY_SIGNED)
```

Next, you can test with bitcoin-cli with `testmempoolaccept '''["<ext_tx_hex>"]'''`.

### Inheritance spend

Try the inheritance key spend as an exercise. You would want to set the correct `nSequence` in order to spend it.
