---
title: "Bitcoin Transactions - Part 3: Signing and generating full transaction "
date: 2022-02-23T16:49:26-08:00
---

In [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}), we learned about keys and addresses and generated our first identity or account. We also transferred some bitcoins on Testnet using a Faucet application. At the end of the session, we have 0.0001 BTC in transferred to us. 

In [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}), we generated a second identity or account, and began to construct our own transaction, a Pay to Public Key Hash (P2PKH) transaction to transfer the 0.0001 BTC from our first account to our second account. 

Recall the rule for when constructing the transaction for signing At the end of Part 2, we added the previous transaction output's locking script in place of where the digital signature as placeholder, so that we can sign the transaction to generate the digital signature. 

### It's coding time! 

No preamble today. We'll dive straight into the code and explain things along the way. 

This is part 3 of a multi-part deep dive into bitcoin transaction. If you want to start from the beginning, go to [Part 1]({{< ref "1-bitcoin-pt1-keys-and-addresses.md" >}}) and follow along. 

We will start from where we left off in Part 2. You can clone the code with the command below: 
```
git clone -b 2-p2pkh-tx https://github.com/yugomike/btc-deepdive.git
```

#### Generating the Digital Signature

Recall in [Part 2]({{< ref "2-bitcoin-pt2-p2pkh-tx.md" >}}), we started to construct a P2PKH transaction. By the end, we had gotten the transaction to the point where it was ready for signing to produce the digital signature. 

To sign the transaction, we will call upon the mighty *helper.py* for the sign() function. A lot of math goes into it, but the concept is fairly simple. You can tell just by looking at the first line: 

```python
def sign(secret_key: int, message: bytes) -> Signature: 
```

You pass in the private key and the message for signing (i.e. the transaction we generated in Part 2) as inputs and the sign() function produces the digital signature as ouput, in the form of a Signature object. This object contains 2 values, the "r value" and the "s value", commonly displayed as (r, s). Together, they represent the digital signature. 


```python
from helper import sign 

# generate the signature 
sig = sign(secret_key, message)
print("The digital signature:", sig)

```
    The digital signature: Signature(r=55271467154578096220369710841914904057772566832324898637955049961746126809827, s=47450177118417380149179574799072871947668868391768804899075684140560698939588)

**Note: your (r, s) will be different from mine since there is a random element in the generation process. This also means your script_sig and tx_id will be different from mine, as the digital signature is an input for them.**

We now have the (r, s) values of the digital signature, but we are not done. Bitcoin requires the transaction to encode the digital signature using Distinguished Encoding Rules or DER encoding. DER encoding is commonly used for the transmission of the digital signature. You can read how it works on the [Bitcoin documentation BIP_0062](https://en.bitcoin.it/wiki/BIP_0062#DER_encoding). 

I've already implemented this functionality into the Signature object as the encode() function, so you just need to call it to generate the DER encoded output. 


```python
# encode the signature as DER encoding 
sig_bytes = sig.encode()
print("The DER encoded digital signature:", sig_bytes.hex())
```

    The DER encoded digital signature: 304402207a328ab345d579e29484b8753558dcd5f83bd06ac8a5f59660836c4e4a6b2ae3022068e7d938303bd420390984181c5fe2d296733fa8132c036e08d277ab531198c4


#### Create the script_sig 
The script_sig is the unlocking script used unlock the 0.0001 BTC UTXO that was sent to us from the Testnet Faucet transaction. It consists of tw parts, the DER encoded digital signature that we just generated and the unhashed public key for identity #1. 

We have just generated the DER encoded digital signature ```sig_bytes```, and we already have calculated the value of the public key in Part 1 (stored in the variable ```public_key```). All we need to do know is to concatenate them them together to form the script_sig. 

This is a simple encoding that chains both items and appends a one-byte suffix, the SIGHASH flags. This is a one-byte suffix that goes into the end of the script_sig. This is a seldom used feature of Bitcoin that lets you create more complex transactions. 

From [Bitcoin's SIGHASH flags documentation](https://wiki.bitcoinsv.io/index.php/SIGHASH_flags): *the mechanism provides a flexibility in constructing transactions. There are in total 6 different flag combinations that can be added to a digital signature in a transaction. Note that different inputs can use different SIGHASH flags enabling complex compositions of spending conditions.*

Fortunately for us, this feature not often used, so that nearly all Bitcoin transactions have the default flag, SIGHASH_ALL. This is what I've implemented in the create_script_sig() function in *helper.py*. 

Once you call this function and give it the digital signature and public key, it outputs the script_sig. 


```python
from helper import create_script_sig 

# generate the script_sig (DER encoded signature + public key)
script_sig = create_script_sig(sig_bytes, public_key) 
print("The script_sig:", script_sig.encode().hex())
```

    The script_sig: 6a47304402207a328ab345d579e29484b8753558dcd5f83bd06ac8a5f59660836c4e4a6b2ae3022068e7d938303bd420390984181c5fe2d296733fa8132c036e08d277ab531198c40121024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6

We finally completed constructing the **script_sig**! All we need to do now is incorporating it into the tranaction input (tx_in) and we have encode the entire transaction. 


```python
# adding script_sig to the transaction input 
tx_in.script_sig = script_sig

# print the full transaction as byte 
print("Completed transaction (in bytestring):", tx)
```

    Tx(version=1, tx_ins=[TxIn(prev_tx=b'g\x07\xaf\\mRW\x06|\x96\x9f\xcf\x7f\x87^j\xd9\xad1C\xe3\x02_\x8c9\x16\x83\xb2<\xff\x9c$', prev_index=1, script_sig=Script(cmds=[b'0E\x02!\x00\x8a\x96\xc0\xc1\xd0\x97\x0c\xec\xefX)\xf9\x17?\xe1\xeaN\xc4\x8a\x9a{\xb9\x03K\xe3\x1c\x8383/\xfe\xff\x02 tO\x9b\xf8\x12\xfae5\xf5\xbf\xdd1\xe6M\xaa+\x11\x11\xc74\xa3\xf2\xd7\x9e\xe9m0\x9a\xd8\xd7\x85L\x01', b'\x02Nv\xf0\x1b\xc8\xad+\x0c\xa7u\xee\x0e9/R\xf5\xdd)\xe7y8\x8ce\x03\x04E\x92\xc5oi\xbf\xe6']), sequence=4294967295)], tx_outs=[TxOut(amount=75000, script_pubkey=Script(cmds=[118, 169, b'\x00\xf6s\x9d^\x8b@\x17\xa9\xee\xbeA2I\xed9I\xe6^$', 136, 172])), TxOut(amount=22000, script_pubkey=Script(cmds=[118, 169, b'6;\xb1\xef\x1d\x87\x91\xbd\xbd~t\x92\xef\x91\xde\xcc\x1e\xb7)]', 136, 172]))], locktime=0)


This is the full transaction that we spent 3 sessons creating! 

Another way to represent this is in hexadecimal:

    Completed transaction (in hex): 0100000001249cff3cb28316398c5f02e34331add96a5e877fcf9f967c0657526d5caf0767010000006a47304402207a328ab345d579e29484b8753558dcd5f83bd06ac8a5f59660836c4e4a6b2ae3022068e7d938303bd420390984181c5fe2d296733fa8132c036e08d277ab531198c40121024e76f01bc8ad2b0ca775ee0e392f52f5dd29e779388c6503044592c56f69bfe6ffffffff02f8240100000000001976a91400f6739d5e8b4017a9eebe413249ed3949e65e2488acf0550000000000001976a914363bb1ef1d8791bdbd7e7492ef91decc1eb7295d88ac00000000

After all the work, it may seem anticlimatic to get a bunch of indecipherable gibberish or a long string of numbers and letters as our output, but as we'll see shortly, this is exactly what we want. This is the raw transaction, without any bells and whistles. It is what the computer sees, stores and transmits acorss the Bitcoin network. 

One final step, we need to generate the transaction id. On the Bitcoin network, every transaction has a unique id. This id is not randomly generated unique id like in many database applications. As you may have learned by now, almost everything in Bitcoin is deterministic, and the id is no different. The transaction id is a hash (surprise surprise!) of the entire transaction. To be more specific, the hashing function used is a double-SHA-256, meaning something like: ```SHA-256(SHA-256(transaction))```.

```python
from helper import generate_tx_id

# once tx goes through, this will be its id (double SHA-256 hash of the transaction)
tx_id = generate_tx_id(tx)
print("Transaction id:", tx_id) 

# also helpful to verify the transaction size
print("Transaction size (bytes): ", len(tx.encode())) # should be 225 bytes

```

    Transaction id: 24076a532fc3325738cf11d63dd5a8d96b7804740d463e9c2a0121e176340df2
    Transaction size (bytes):  225

Voila! We have created our transaction and gave it a transaction id. We can now propagate this into the Bitcoin network! And we will...in Part 4.


### A note about transaction encoding and serialization

For the rest of Part 3, I want to spend more time understanding what we've just created, and also talk about some important concepts, in particular, **encoding**, **serialization** and **endianness**. 

#### Encoding & serialization
A transaction takes on many forms depending on the intended audience. 

When you see a transaction on a block explorer website, such as our [Testnet Faucet transaction](https://www.blockchain.com/btc-testnet/tx/6707af5c6d5257067c969fcf7f875e6ad9ad3143e3025f8c391683b23cff9c24) from earlier, it is beautifully formatted with additional information presented. The inputs and outputs are shown as well as their correspoinding addresses. It even goes a step further and show you the commands behind the op_codes. This is a human-readable form. A lot of work is done behind the scene by the software on those blockchain websites to make the transaction look like this. Certain information have to be retrieved ahead of time or computed. 

In contrast, in its most original form, a Bitcoin transaction has very few information and looks like this:

    Tx(version=1, tx_ins=[TxIn(prev_tx=b'g\x07\xaf\\mRW\x06|\x96\x9f\xcf\x7f\x87^j\xd9\xad1C\xe3\x02_\x8c9\x16\x83\xb2<\xff\x9c$', prev_index=1, script_sig=Script(cmds=[b'0E\x02!\x00\x8a\x96\xc0\xc1\xd0\x97\x0c\xec\xefX)\xf9\x17?\xe1\xeaN\xc4\x8a\x9a{\xb9\x03K\xe3\x1c\x8383/\xfe\xff\x02 tO\x9b\xf8\x12\xfae5\xf5\xbf\xdd1\xe6M\xaa+\x11\x11\xc74\xa3\xf2\xd7\x9e\xe9m0\x9a\xd8\xd7\x85L\x01', b'\x02Nv\xf0\x1b\xc8\xad+\x0c\xa7u\xee\x0e9/R\xf5\xdd)\xe7y8\x8ce\x03\x04E\x92\xc5oi\xbf\xe6']), sequence=4294967295)], tx_outs=[TxOut(amount=75000, script_pubkey=Script(cmds=[118, 169, b'\x00\xf6s\x9d^\x8b@\x17\xa9\xee\xbeA2I\xed9I\xe6^$', 136, 172])), TxOut(amount=22000, script_pubkey=Script(cmds=[118, 169, b'6;\xb1\xef\x1d\x87\x91\xbd\xbd~t\x92\xef\x91\xde\xcc\x1e\xb7)]', 136, 172]))], locktime=0)


Yikes! 

This is the transaction we just constructed (actually, it wouldn't even have any of the attributes like version and TxOut. It would just be gibberish with all the backslashes). 

This raw format is how all transaction data are stored and transmitted on the Bitcoin network. It is in the form of bytestring, essentially ASCII representation of the transaction data, that is efficient for computer systems to process, but basically unredable for human. This is a process known as serialization, turning objects and data into streams of bytes so that they can be stored on disk or sent over the network. 

The first step from gibberish to human-readable form is to encode the bytestring into a different, more readable format. Encoding is the process of converting data from one format to another, in this case, from bytestring to hexadecimal or hex. 

Hex is popular for storing data because 2 hexes = 1 byte. Recall that hex is a base 16 numerical system and each hex is represented by 4 bits. Since each byte is 8 bits, a byte can be represented precisely by 2 hexes. You may have noticed during the coding walkthrough that we included ```.hex()``` in many of our outputs so far. This converts bytestrings into more readable hex format. 

After encoding the raw transaction data into hex, it would look like this (note I added a space after every 2 hex to make each byte more distinguishable): 

```
01 00 00 00 01 24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67 01 00 00 00 6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6 ff ff ff ff 02 f8 24 01 00 00 00 00 00 19 76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac f0 55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac 00 00 00 00
```

This may look like a string of seemingly unrelated numbers, but with a little bit of practice and familiarity, you can actually read this like a transaction. 


#### Endianness 
Finally, there is something called endianness. Don't let the fancy word fool you. It just means the order the system stores and transmit byte data. It can either store the most significant digit first (big-endian) or the least significant digit first (little-endian).

The best way to understand endianness is through an example. Take a random number, say 1234. 

When you read this number out loud, you will say “one thousand two hundred and thirty four”. You start with the most significant value, 1000, and end with the least significant value, 4. This is known as big-endian (i.e. the big end). Little endian, in contrast, operates backward, working your way up from the least significant value to the most significant value. 

In conversations, big-endian makes sense. Saying one thousand automatically implies there are 4 digits. But when computers transmit data, it is done one value at a time (one byte at a time to be more precise), so there is no way to know in advance how many total values you are receiving. Therefore, computers generally process data from the least significant value first, and this also applies to the Bitcoin protocol. 

When we start to decipher the raw transaction, to get the true value of some of the information, we will have to convert the little-endian to big-endian. This is as simple as reversing each byte (2 hexes). So ```8a ab 84 53``` in little-endian is actually ```53 84 ab 8a```. 

#### Pop Quiz

At this point, having generated the transaction from scratch. A good test of your knowledge is to take the transaction in hex format, and see if you can break it apart into its components. 

```
01 00 00 00 01 24 9c ff 3c b2 83 16 39 8c 5f 02 e3 43 31 ad d9 6a 5e 87 7f cf 9f 96 7c 06 57 52 6d 5c af 07 67 01 00 00 00 6a 47 30 44 02 20 54 94 4a 0b 19 5f fb b1 d8 84 58 37 5a 29 c3 1f bc 46 fd b7 cc 5a 9b ef a9 9d 07 b3 8e e0 0e 10 02 20 0d a2 98 a2 8c 5f 1f 90 d2 e8 ef b6 25 f8 17 60 8e cc 3d 6b 16 da 57 f6 cb fb 46 00 8a ab 84 53 01 21 02 4e 76 f0 1b c8 ad 2b 0c a7 75 ee 0e 39 2f 52 f5 dd 29 e7 79 38 8c 65 03 04 45 92 c5 6f 69 bf e6 ff ff ff ff 02 f8 24 01 00 00 00 00 00 19 76 a9 14 00 f6 73 9d 5e 8b 40 17 a9 ee be 41 32 49 ed 39 49 e6 5e 24 88 ac f0 55 00 00 00 00 00 00 19 76 a9 14 36 3b b1 ef 1d 87 91 bd bd 7e 74 92 ef 91 de cc 1e b7 29 5d 88 ac 00 00 00 00
```

We will walk through this example together at the start of Part 4. Stay tuned! 