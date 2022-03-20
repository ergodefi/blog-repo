---
title: "Developing Bitcoin Intuition - Transactions: Part 2"
date: 2022-02-18T20:56:53-08:00
---

## Constructing a Bitcoin Transaction 

Let's first highlight the difference between a real life transaction and a bitcoin transaction. Knowing the difference will help you understand bitcoin better. In real life, if you buy a pizza for \$20.25 and you hand the cashier \$25, you will get \$4.75 back. 

In Bitcoin, that same transaction would look more like this: you give all the cash inside your wallet to the cashier (let's assume there is $100 in your wallet). Then you send back \$79.70, leaving \$20.30. The cashier keeps \$20.25 for the cost of the pizza, and the remaining \$0.05 goes to the miner as transaction fee. 

## Primer on bitcoin transaction

### Unspent Transaction Output or UTXO

In Bitcoin, there is no free-form representation of money like bills and coins; instead, all funds are represented as unspent outputs from earlier transactions. In Bitcoin lingo, these are called Unspent Transaction Outputs (UTXO). Whenever you transact with bitcoins, you have to use the entire value associated with the an account. One or more of the outputs would go to recipients of the transaction as intended, and there is usually also one output that returns the extra "change" back to one of your own accounts. 

What happens if there's not enough funds in an address? You would simply add additional inputs, which would in effect consolidate funds from multiple accounts to fund a single large transaction. At the end of the day, bitcoin transactions are just a different way of thinking about money. By altering the inputs and outputs based on the funds that you have access to, you can construct any transaction on the Bitcoin network. 

### Transaction fees

Every Bitcoin transaction has a cost to it. This is the fee awarded to the miner who added your transaction to the block they are creating for the Bitcoin blockchain. The fee serves an important function both as payment for the miner (i.e. who exchanged electricity and processing power to create the block) and incentive to include your transaction in the block (i.e. instead of someone else's transaction). 

The fee is a very small amount that is calculated based on the complexity of the transaction (number of inputs and outputs) rather than the amount being transferred. This amount is not explicitly stated in the transaction as an output, but is instead computed by subtracting the total value of outputs from inputs. We'll cover the creation of blocks and formation of blockchains in a separate series, so for the purpose of understanding transactions, just know that this fee exists and it is awarded to the miner. 

### Locking and unlocking scripts
As you can see, a Bitcoin transaction is nothing more than inputs and outputs, from which complex transactions can be built. Inputs are essentially outputs from earlier transactions that are will be spent if the transaction is successful. The outputs are the unspent transaction outputs available )(UTXO) which are available to be used as inputs in future transaction. In this sense, each transaction input is simply referencing a prior tranasction output. 

Bitcoin uses the Ellptical Curve Digital Signature Algorithm (ECDSA) and public-key encryption to keep the UTXOs secure so that only the intended recipients can access the funds. These security meaures are built directly into the transaction. 

The most popular form of Bitcoin transaction, between 2 people, is called a **pay-to-public-key-hash (P2PKH)** transaction. In this type of tranasction, every input contains an unlocking script called **script_sig**, which is made up of the digital signature and its corresponding public key. This unlocking script is used to unlock the UTXO from the prior transaction. Similarly, every output (UTXO) has a locking script called **script_pubkey** that can only be opened by the unlocking script on a corresponding transaction input. 

The locking and unlocking analogy is perfectly appropriate here. The locking script is basically a lock that can only be opened by a particular the unlocking script, or a key. When you attempt to use the key, if it is successful, the UTXO is consumed. Otherwise, the transaction is nullified and no exchange of value happens. As we shall see later, this is done using Bitcoin's own scripting language, appropriately named **Script**. 

In Part 3, we will write the Python code to generate and verify the digital signature. But before we get to the digital signature, we need to generate all the other components of a transaction. Let's do this now. 

## It's coding time! 

Now that we have some bitcoins and learned about how transaction works, let's create our own transaction. We are going to create one of the most common transactions: paying for something and getting some change back. 

If you want to follow along from the beginning, start with [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}). Alternatively, you can clone the code from Part 1 using the command below. 

```
git clone -b 1-wallet-address https://github.com/yugomike/btc-deepdive.git 
code btc-deepdive
```

### Generating the second identity

Since a transaction requires at least 2 parties: one party to send and one party to receive, let's create our second identity now. We can do this easily using the functionality we have already seen in Part 1. 

```python
# identity 2 
# ========================
secret2 = int.from_bytes(b'another secret identity two', 'big')
privateKey2 = PrivateKey(secret2)
publicKey2 = privateKey2.point
```

Same as before, we'll create a static private key for illustration purpose, but never do this with a real Bitcoin transaction! Once we create the PrivateKey object, getting the rest of the information is quite easy since the functionalities are already built in. 

```python
print("Bitcoin Identity #2")
print("* Private key: ", privateKey2.secret)
print("* Public key (Point): ", (publicKey2.x.num, publicKey2.y.num)) 
print("* Public key (SEC Compressed): ", publicKey2.sec().hex())
print("* Public key (SEC Uncompressed): ", publicKey2.sec(compressed=False).hex())
print("* Public key hash: ", publicKey2.hash160().hex())  
print("* Bitcoin address ", publicKey2.address(testnet=True))
```

```output
Bitcoin Identity #2
* Private key:  40080948312511263619633009066092429669818823132623907411220264815
* Public key (Point):  (110398465478409316276920527699961004613373518058286253761163568467326599764908, 27581862894942929806511524173629706827067517406769995130296969522417828343338)
* Public key (SEC Compressed):  02f413512fca28fd30175ac631e3d5836dc0caac14d28236bacad57a9db7b11bac
* Public key (SEC Uncompressed):  04f413512fca28fd30175ac631e3d5836dc0caac14d28236bacad57a9db7b11bac3cfac7faf9306712a71d21228bc4cd0726d066c6a69cc1520f67c9a6584fae2a
* Public key hash:  00f6739d5e8b4017a9eebe413249ed3949e65e24
* Bitcoin address  mfc3Xp6SfPZq2AfgXcVWRoZzWrqjZQtCrp
```


### Step 1: Create the trasnsaction input 

Since a transaction is made up of inputs and outputs, to construct the tranasction, we start by generating these elements (known as Transaction Inputs and Transaction Outputs or TxIn and TxOut respectively). In the transaction we are creating, there is 1 input and 2 outputs. 

Let's start with the input. Every input has 4 elements that must be included in the transaction: (1) id previous transaction; (2) index of the output in the previous transaction; (3) the script_sig or unlocking script and (4) the sequence. 

(1) and (2) combined will tell Bitcoin exactly which output from the earlier transaction we want. (3) the unlocking script is used to unlock the locking script on the UTXO. This script is made up of the digital signature, signed by the account owner's private key and the public key. I will show you the entire unlocking process in Part 3. (4) is a feature that is currently disalbed in Bitcoin, and always has the hex value `0xFFFFFFFF`. 

Let's construct this input. 

```python
from tx import TxIn, TxOut, Tx
from script import p2pkh_script, Script 
from helper import SIGHASH_ALL

# Transaction 
# =====================
# 
# STEP 1:   Create TxIn objects to store transaction inputs 
tx_in = TxIn(
    prev_tx = bytes.fromhex('6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24'),
    prev_index = 1
    # The script_sig cannot be created yet 
)
```
First, we have to import some useful classes. These classes will take care much of the work in the background for us, so that we can just focus on building our mental model. 

Recall in [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}), we asked the Testnet Faucet for some tBTCs. This triggered a bitcoin transaction, sending 0.0001 tBTC to our identity #1. That output is the input for this transaction. Here the info in case you forgot:

* Prev tx id: `6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24`
* Index: `1` because it is the second output (index starts at `0`) 

We cannot create the script_sig yet, since the digital signature is generated by signing the entire transaction, which we are still building. We will come back to this. Finally, since the sequence is disabled, it is always `0xFFFFFFFF`, and this has been built into the TxIn class already. 

### Step 2: Create the transaction outputs 

Every transaction output has 2 elements: (1) the amount of the output in satoshis, the smallest denomination in Bitcoin (1 BTC = 1,000,000 satoshis); and (2) the script_pubkey, also known as the locking script. 

We are creating 2 outputs. The first is for 75000 satoshis, which will be going to our newly created identity #2 wallet. The second is for 22000 satoshi, going back to our identity #1 wallet as change. This means 3000 satoshis will be going to the miner. 

Let's see how this works. 

```python

from script import p2pkh_script, Script 

# STEP 2:   Create TxOut objects to store transaction outputs 
# Create output #1 object
tx_out1 = TxOut(
    amount = 75000,
    script_pubkey = p2pkh_script(publicKey2.hash160()) 
    # OP_DUP OP_HASH160 00f6739d5e8b4017a9eebe413249ed3949e65e24 OP_EQUALVERIFY OP_CHECKSIG
)  

# Create output #2 object
tx_out2 = TxOut(
    amount = 22000,
    script_pubkey = p2pkh_script(publicKey.hash160()) 
    # OP_DUP OP_HASH160 363bb1ef1d8791bdbd7e7492ef91decc1eb7295d OP_EQUALVERIFY OP_CHECKSIG
)
```

We created two `TxOut` objects, one for each output. I've included a comment below each output with what the script_pubkey (locking script) looks like, but they won't make sense to you right now, and that'sokay. I will explain what those OP_CODES do, as well as how the unlocking mechanism work in [Part 3]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md" >}}). For now, you just need to know that the locking script contains the public key hash of the recipient's wallet. This explains why output #1 has the public key hash of identity #2. 

### Step 3: Create the transaction 

We have a partial input created (missing the script_sig) and the outputs fully created. We are ready to sign the transaction and generate the digital signature. Before we do that, let's create a new object that combines everything we've constructed. I've called it Tx, short for transaction. 

```python
# STEP 3 - Create the transaction object to consolidate the info 
tx = Tx(
    version = 1,
    tx_ins = [tx_in],
    tx_outs = [tx_out1, tx_out2],
    locktime = 0,
    testnet=True
)
```

Remember I said that a transaction is made up primarily of inputs and outputs? There are a few other minor elements that we need to include in a transaction. First, there is the `version`. Bitcoin is currently on version 1 for transactions, but in the future newer versions may be introduced. 

Second, there is `locktime`, which is a rarely used feature that allows Bitcoin to lock a transaction until a future date, similar to a postdated cheque. We will see later that this feature is not used because there is a superior way to create a postdated transaction using a different mechanism, called Pay-to-Script-Hash (P2SH). We will cover this later. For now, we will use the default value, which is 0. 

Finally, for our implemention, we also used a Boolean value to determine whether this transaction is being created on `Testnet` or the real Bitcoin network. This is because the Bitcoin protocol differentiates between the two by using different prefixes. This is so that you don't end up accidentally deploying your transaction in the wrong place. Since we are building on Testnet, we set this to True. 



### Step 4 - Sign the transaction to generate the signature

We are now finally ready to sign the transaction. This will create the digital signature. From there, we can construct the `script_sig` and therefore the entire the transaction. 

**What is a digital signature anyway, and why do we need this?** 

A P2PKH transaction is simply where person #1 sends bitcoins to the bitcoin address of person #2 (and person #3, #4, #5 etc. depending on the number of outputs). 

What is a bitcoin address? It is nothing more than the Base58Check encoding of the public key hash, which is itself derived from the public key. And where does the public key come from? The private key of course! Therefore, ultimately the private key creates the bitcoin address. This is done deterministically through one-way functions. One private key always has one and the same bitcoin address. 

But how do we know whether person #2 has the private key? Person #2 cannot disclose the private key of course, but he can give you a digital signature created by the private key. The digital signature allows a person to demonstrate possession of a private key without disclosing it. 

In Bitcoin, the digital signature is created by hashing the entire transaction twice using the SHA256 hashing function. During signing, the Bitcoin protocol will replace the `script_sig` (which we don't have yet) with the `script_pubkey` from the output (which we created in Step 2). The resulting hash is then signed to generate the digital signature. The ECDSA digital signature comes in the form of 2 values, `r` and `s`. Collectively, they are often displayed as `(r, s)`. 

Below is the code we used to generate the `(r, s)` values of the digital signature: 

```python
# STEP 4 -  Sign the transaction to generate the digital signature 
z = tx.sig_hash(0) 
rs_values = privateKey.sign(z)

print("r:", rs_values.r)
print("s:", rs_values.s)
```
The `sig_hash()` function constructs a temporarily a transaction with the information we collected in Steps 1 to 3, substituting `script_sig` with the `script_pubkey`. This is then hashed using 2 rounds of SHA256. We store the resulting hash in variable `z`, which is then passed into the `sign()` function to generate the signature (r, s) values. 

```output 
r: 43600387006341076137948396205293412800582818280246568097523522344188730004800
s: 4357887215047044106015785372445069492339273917811775633768528333880651902141
```

But wait. We are not done. Even though we have the `(r, s)` digital signature values, it'll take a few more steps to create the `script_sig`. I'll show you the code first, then explain what each line does. 


```python
der = digital_signature.der() # DER encoded signature
sig = der + SIGHASH_ALL.to_bytes(1, 'big') #SIGHASH flag 
script_sig = Script([sig, privateKey.point.sec()]) # script_sig = sig + public key

print("DER signature:", der.hex())
print("script_sig:", script_sig)
```

```output
DER signature: 304402206064f1cc900e947012b6a725525d608747723aa18391dd8ad67ddc9427a9d540022009a27a0c9edc15d1713407125433d516ad233216e509d77dde33345dc3af6cbd
script_sig: 304402206064f1cc900e947012b6a725525d608747723aa18391dd8ad67ddc9427a9d540022009a27a0c9edc15d1713407125433d516ad233216e509d77dde33345dc3af6cbd01 024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6
```

First, Bitcoin protocol formats the digital signature in a particular way, called the Distinguished Encoding Rules (DER). This is an encoding method that uses variable integers (varints) to tell Bitcoin how many bytes it should expect. We will talk about more encoding in [Part 3]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md" >}}). For now, we'll just implement the DER encoding in Python. 

Next, we have to append an extra value onto the DER encoded signature. This is a flag known as `SIGHASH_FLAG`. This flag is used to indicate which part of the transaction is signed by the digital signature. Refer to this [page](https://wiki.bitcoinsv.io/index.php/SIGHASH_flags) for the different flags that exist. We will use the default flag value, which is `1`. It means `SIGHASH_ALL` or sign all inputs and outputs. 

When you the `DER encoded signature` is appended with the `SIGHASH_FLAG`, that combines to form the `sig` or signature. 

Finally, `script_sig` is the signature plus the compressed public key of the account that created the transaction (in this case, that is identity #1). We already have this from Part 1: `privateKey.point.sec()`. 


### Step 5 - Put the transaction together 

The `script_sig` was the last piece of the puzzle. All we have to do now is add the script_sig into the transaction object `tx` we created in Step 3 to complete our transaction. You can see all of the elements of the `tx` object, which contains the transformation information we collected in all of the earlier steps. 


```python
tx.tx_ins[0].script_sig = script_sig  # incorporate script_sig into transaction 
print(tx)
```

```output
version: 1
tx_ins:
6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24:1
tx_outs:
75000:OP_DUP OP_HASH160 00f6739d5e8b4017a9eebe413249ed3949e65e24 OP_EQUALVERIFY OP_CHECKSIG
22000:OP_DUP OP_HASH160 363bb1ef1d8791bdbd7e7492ef91decc1eb7295d OP_EQUALVERIFY OP_CHECKSIG
locktime: 0
```

With the transaction created, let's call it a day for this part. We have accomplished a lot today, generating a second wallet and creating a full Bitcoin transaction from scratch. In [Part 3]({{< ref "3-bitcoin-pt3-script-sig-full-tx.md" >}}), we will round out our knowledge by taking this transaction apart to show you how Bitcoin transactions are transmitted in the Bitcoin's peer-to-peer network. Along the way, we'll pick up some very useful computer science knowledge. 

Here is a sneak preview of what is to come in Part 3. 

```python
print(tx.serialize().hex()) # this is our transaction
```
`0100000001249cff3cb28316398c5f02e34331add96a5e877fcf9f967c0657526d5caf0767010000006a47304402206064f1cc900e947012b6a725525d608747723aa18391dd8ad67ddc9427a9d540022009a27a0c9edc15d1713407125433d516ad233216e509d77dde33345dc3af6cbd0121024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6ffffffff02f8240100000000001976a91400f6739d5e8b4017a9eebe413249ed3949e65e2488acf0550000000000001976a914363bb1ef1d8791bdbd7e7492ef91decc1eb7295d88ac00000000`

See you then!
