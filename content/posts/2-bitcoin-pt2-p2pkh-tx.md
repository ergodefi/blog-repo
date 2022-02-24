---
title: "Bitcoin Transactions - Part 2: generating a P2PKH transaction"
date: 2022-02-19T20:56:53-08:00
---

Before constructing a bitcoin transaction, let's first highlight the difference between a real life transaction and a bitcoin transaction. Knowing the difference will help you understand bitcoin better.

In real life, if you buy a pizza for \$20.25 and you hand the cashier \$25, you will get \$4.75 back. 

On the Bitcoin network, and assuming there is $100 in your account, it would look like this: you give all the cash inside your wallet to the cashier and then you take back \$79.70, leaving \$20.30. The cashier keeps \$20.25 for the cost of the pizza, and the remaining \$0.05 goes to the miner as transaction fee. 

### Primer on bitcoin transaction

#### Unspent Transaction Outputs or UTXO 

You see in Bitcoin, there is no freeform representation of money like a $5 bill; instead, all funds are represented as unspent outputs from earlier transactions or UTXOs. Whenever you transact with bitcoins, you have to use the entire value associated with the an account. One or more of the outputs would go to recipients of the transaction as intended, and there is usually also one output that returns the extra "change" back to one of your own accounts. 

What happens if there's not enough funds in an address? You would simply add additional inputs, which would in effect consolidate funds from multiple accounts. 

At the end of the day, bitcoin transactions are just a different way of thinking about money. By altering the inputs and outputs based on the funds that you have access to, you can construct any transaction on the Bitcoin network. 

#### Locking and unlocking scripts
As you can see, a bitcoin consists primarily consists of inputs and outputs. The inputs are essentially outputs from earlier transactions  that are will be spent if the transaction is successful. And the outputs are the UTXOs, unspent transaction outputs available for a future transaction. 

To keep the UTXOs secure, so that the intended recipient has to demonstrate they have the private key, Bitcoin uses a clever locking and unlocking scripts using digital signature (the elliptical curve variety). 

The way it works is that when the transaction outputs are created, they come with a locking script that locks the UTXO so that only the person possessing the right private key can solve the algorithmic puzzle. This puzzle is created using Bitcoin's own scripting language, called Script. 

When you create the transaction that consumes the UTXO, you use your private key to sign the transaction to create a digital signature (known as script_sig). The script_sig is encoded along with the input. If you use the correct private key to create the digital signature, you will solve the puzzle, thereby unlocking the UTXO for consumption. If you used the wrong private key, the puzzle fails and the transaction is nullified. 

There is a lot of math in the verification of the digital signature. If you want to learn the math, I recommend reading [Chapter 6](https://github.com/bitcoinbook/bitcoinbook/blob/develop/ch06.asciidoc) of Andreas Antonopoulos' excellent book, Mastering Bitcoin. 

In Part 3, we will write the Python code to generate and verify the digital signature. But before we get to the digital signature, we need to generate all the other components of a transaction. Let's do this now. 


### It's coding time! 

Now that we have some bitcoins and learned a bit about how transaction works, let's go ahead and create our own transaction! We are going to create one of the most common transactions (in Bitcoin and real life), which is paying for something and getting some change back. 

This is part 2 of a multi-part deep dive into bitcoin transaction. If you want to follow along, you should start with [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}). Alternatively, you can clone the code from Part 1 using the command below.

```
git clone -b 1-wallet-address https://github.com/yugomike/btc-deepdive.git
```



#### Generating the second identity

First, we'll need a second account (i.e. a new set of private and public keys) so that we can transfer the bitcoin from one account to another and observe the entire process.  We can do this easily using the functionality we have already seen in Part 1, made available by *helper.py*. 


```python
# identity 2 
# ========================
secret_key2 = int.from_bytes(b'another secret identity two', 'big') # for reproducibility 
assert 1 <= secret_key2 < bitcoin_gen.n
public_key2 = secret_key2 * G
public_key2_compressed = PublicKey.from_point(public_key2).encode(compressed=True, hash160=False).hex()
public_key_hash2 = PublicKey.from_point(public_key2).encode(compressed=True, hash160=True).hex()
bitcoin_address2 = PublicKey.from_point(public_key2).address(net='test', compressed=True)

print("Bitcoin Identity #2")
print("* Secret (Private) Key: ", secret_key2)
print("* Public key (uncompressed): ", (public_key2.x, public_key2.y))
print("* public key (compressed): ", public_key2_compressed) 
print("* Public key hash: ", public_key_hash2) 
print("* Bitcoin address: ", bitcoin_address2)
```

    Bitcoin Identity #2
    * Secret (Private) Key:  40080948312511263619633009066092429669818823132623907411220264815
    * Public key (uncompressed):  (110398465478409316276920527699961004613373518058286253761163568467326599764908, 27581862894942929806511524173629706827067517406769995130296969522417828343338)
    * public key (compressed):  02f413512fca28fd30175ac631e3d5836dc0caac14d28236bacad57a9db7b11bac
    * Public key hash:  00f6739d5e8b4017a9eebe413249ed3949e65e24
    * Bitcoin address:  mfc3Xp6SfPZq2AfgXcVWRoZzWrqjZQtCrp

#### Generating the  inputs and output (TxIn and TxOut) objects
Second, we need to generate the inputs and outputs (known as TxIn and TxOut respectively). We also need a Script object to store the locking and unlocking scripts, which are used in the locking and unlocking of the UTXO. Finally, we will construct the transaction (Tx) itself. Again, I have written the functionality in *helper.py*, so we can simply focus on understanding the process. 


```python
from helper import TxIn, TxOut, Script, Tx

# transaction input #1 
tx_in = TxIn(
    prev_tx = bytes.fromhex('6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24'), 
    prev_index = 1, # the 2nd output 
    script_sig = None, # signature to be inserted later 
    sequence = 0xffffffff, # almost never used and default to 0xffffffff
)
```

Now let's construct the transaction input. There is just 1 input, which is the funds we received from the Testnet Faucet transaction from [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}). 

That was a transaction with its own inputs and outputs, and one of its outputs was 0.0001 BTC sent to our public hey kash address. That was an example of an UTXO, and we are now going to spent it in the transaction we are creating. 

To identify this input, we reference the previous transaction's address (in the form of the public key hash) and the the index of that output (1 is he index as it indicates it was the second output). You don't have to specify the actual value of the input, as Bitcoin can simply look up that previous transaction for the output's value. 

We also need to generate the digital signature (script_sig) used to unlock the UTXO. However, we cannot generate this yet, as it involves signing the entire transaction, which we are still constructing. We will come back to this later. 

Finally, every input also contains a sequence. It is not a feature currently being used, and this is indicated by the Bitcoin protocol as 0xffffffff. 

#### Transaction Outputs 
We will create 2 outputs, and this means we need 2 instances of TxOut. Unlike with input, we have to specify the actual amount for each output. Note that the amount is stored in the smallest unit of bitcoin, the satoshi, and 1 bitcoin = 100,000,000 sats. The transaction fee paid to the miner is not explicitly stated in the transaction. It is simply the difference between the sum of inputs and outputs. 


```python
# transaction output #1
tx_out1 = TxOut(
    amount = 75000,
)

# transaction output #2
tx_out2 = TxOut(
    amount = 22000,
)

# 75000 + 22000 = 97000, which means 3000 sats are paid to the miner as transaction fee 
```

We also have to create the locking script (or script_pubkey) for each output. The reference to pubkey refers to the public key hash. Each output is constructed so that it can only be unlocked by the person holding the private key for that public key hash. Recall the one-way relationships: private key creates the public key; and the public key creates the public key hash. 

This means:
* as output 1 is being transferred to identity #2, its script_pubkey must include the public key hash for identity #2
* as output 2 is transferred back to identity #1, its script_pubkey must include the public key hash for identity #1


```python
output1_pkh = PublicKey.from_point(public_key2).encode(compressed=True, hash160=True)  
output2_pkh = PublicKey.from_point(public_key).encode(compressed=True, hash160=True)

# 118, 169, 136 and 172 are op_codes. # Refer to https://en.bitcoin.it/wiki/Script for more info
output1_script = Script([118, 169, output1_pkh, 136, 172])
output2_script = Script([118, 169, output2_pkh, 136, 172])

output1_script.encode().hex() # output 1 in hex: 1976a91400f6739d5e8b4017a9eebe413249ed3949e65e2488ac
output2_script.encode().hex() # output 2 in hex: 1976a91400f6739d5e8b4017a9eebe413249ed3949e65e2488ac

tx_out1.script_pubkey = output1_script # adding script_pubkey to output 1
tx_out2.script_pubkey = output2_script # adding script_pubkey to output 2
```

#### The placeholder for script_sig

So far, we have created 2 identities, the input and outputs, and the locking scripts for the outputs, known as script_pubkey. 

The final piece of puzzle we need to is the unlocking script, or the script_sig. The unlocking script has 2 parts: (1) the digital signature; and (2) the unhashed public key. 

We generated the public key in Part 1, and now must generate the digital signature. But, there is a problem. 

You see, the digital signature is generated by signing the hash of the entire transaction, which we are still building. So it seems we have a catch 22 on our hands. How can we get generate the digital signature when it requires the entire transaction to do it? 

Fortunately, the Bitcoin protocol has a rule that solves our issue. It requires that we replace the script_sig (which we don't have) with the script_pubkey from the previous transaction's corresponding output (which we can find). 

What is the script_pubkey from the previous transaction? You can find it by going back to the [previous transaction](https://www.blockchain.com/btc-testnet/tx/6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24). With the op_codes expressed, it looks like the highlighted below. 

![img](/images/prev_tx_script_pubkey.png)

Staying true to the nature of this series, we are going to recreate it a more elementary format. 


```python
# retrieve previous transaction output public key hash 
public_key_hash = PublicKey.from_point(public_key).encode(compressed=True, hash160=True)

# constructing the previous tx locking script 
prev_tx_script_pubkey = Script([118, 169, public_key_hash, 136, 172])

# adding the locking script as placeholder for input digital signature
tx_in.prev_tx_script_pubkey = prev_tx_script_pubkey 

print("Previous tx locking script:", prev_tx_script_pubkey.encode().hex())
```

    Previous tx locking script: 1976a914363bb1ef1d8791bdbd7e7492ef91decc1eb7295d88ac

#### Constructing the transaction (without digital signature)

Now we have everything to finally construct the transaction! First, let's import the transaction object (Tx) from *helper.py*. We will be storing all the ingredients inside this object, and use its encode() function to create the message for signing. 

```python
tx = Tx(
    version = 1, # currently just version 1 exists
    tx_ins = [tx_in],
    tx_outs = [tx_out1, tx_out2],
    locktime = 0,
)
# sig_index 0 means crafting digital signature for a specific input index 
message = tx.encode(sig_index = 0) 
print("Message for signing: ", message.hex())
```

    Message for signing:  0100000001249cff3cb28316398c5f02e34331add96a5e877fcf9f967c0657526d5caf0767010000001976a914363bb1ef1d8791bdbd7e7492ef91decc1eb7295d88acffffffff02f8240100000000001976a91400f6739d5e8b4017a9eebe413249ed3949e65e2488acf0550000000000001976a914363bb1ef1d8791bdbd7e7492ef91decc1eb7295d88ac0000000001000000

That's it for Part 2. I was hoping to cover up to the generation of the digital signature, but it is getting too long. So instead, we will pick up from where we left off and generate the digital signature in the next part. We will then finally create the final transaction and propagate it to the Bitcoin network (Testnet). Stay tuned! 